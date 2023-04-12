# Install Odoo on GKE and connect to a Postgres database on Cloud SQL with auth proxy

Bài viết mình có tham khảo từ [guide chính chủ](https://cloud.google.com/sql/docs/postgres/connect-instance-kubernetes). Để chạy các lệnh gcloud ta có thể dùng trực tiếp Cloud Shell trên web hoặc cài gcloud CLI trên terminal.

Truớc khi bắt đầu chúng ta cần enable các API sau:
- Compute Engine API
- Cloud SQL Admin API
- Google Kubernetes Engine API
- Artifact Registry API
- Cloud Build API

## 1. Tạo project trên console Google Cloud và enable billing cho project đó.

## 2. Tạo instance Cloud SQL và database

Trong mục **Connections** thì ta nên chọn **Private IP**, làm theo các bước ở mục **Set Up Connection** để tạo IP. Sau khi tạo được instance, vào mục **Databases** trong navigation menu để tạo mới database. Vào mục **Users** trong menu để tạo mới user, chú ý lưu lại password của user.

## 3. Tạo GKE cluster

Truy cập trang **Google Kubernetes Engine** trên console Google Cloud để tạo cluster. Phần cấu hình mình chọn Autopilot để Google tự tùy chỉnh cấu hình cho chúng ta. Với cluster được tạo theo kiểu Autopilot thì sẽ được tự động enable [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity).

Sau khi tạo cluster (sẽ mất chút thời gian đợi) thì ta chạy lệnh sau để gcloud và kubectl kết nối với cluster vừa tạo, trong đó autopilot-cluster-1 là tên mình cluster vừa tạo.

```
gcloud container clusters get-credentials autopilot-cluster-1 --region asia-southeast1
```

## 4. Set up service account

GKE sẽ cần một service account với role **Cloud SQL Client** để có quyền connect tới Cloud SQL. Đầu tiên, mình dùng lệnh gcloud iam service-accounts create để tạo mới service account có tên gke-service-account.

```
gcloud iam service-accounts create gke-service-account \
  --display-name="GKE Service Account"
```

Chạy tiếp ```gcloud projects add-iam-policy-binding``` để gán role **Cloud SQL Client** cho service account vừa tạo. Phần odoo-lab-vsi là ID của project mình tạo.

```
gcloud projects add-iam-policy-binding odoo-lab-vsi \
  --member="serviceAccount:gke-service-account@odoo-lab-vsi.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```

Sau khi xong bạn vẫn cần tạo thêm k8s service account và bind nó với service account của Google Cloud. Ta tạo file service-account.yaml như sau để tạo service account tên là ksa-bidv-cloudsql.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ksa-bidv-cloudsql
```

Chạy lệnh ```kubectl apply -f service-account.yaml```.

Với lệnh ```add-iam-policy-binding``` ta sẽ bind service account của Google Cloud và k8s với nhau.

```
gcloud iam service-accounts add-iam-policy-binding \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:odoo-lab-vsi.svc.id.goog[default/ksa-bidv-cloudsql]" \
  gke-service-account@odoo-lab-vsi.iam.gserviceaccount.com
```

Cuối cùng dùng lệnh sau để hoàn tất việc bind

```
kubectl annotate serviceaccount \
  ksa-bidv-cloudsql  \
  iam.gke.io/gcp-service-account=gke-service-account@odoo-lab-vsi.iam.gserviceaccount.com
```

## 5. Config secret

Để lưu user và password của database và truyền vào trong file deployment của GKE thì mình tạo secret. Phần username và password điền theo thông tin bạn đã tạo ở bước trước.

```
kubectl create secret generic gke-cloud-sql-secrets \
  --from-literal=database=odoo \
  --from-literal=username=odoo \
  --from-literal=password=password;
```

## 6. Deploy Odoo lên GKE

Trong phần **Workloads** trên menu navigation của GKE có lựa chọn cho mình deploy image từ registry của Google nhưng sẽ hơi khó edit sau khi tạo nên mình dùng file deployment.yaml để deploy.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: odoo-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: odoo-app
  template:
    metadata:
      labels:
        app: odoo-app
    spec:
      serviceAccountName: ksa-bidv-cloudsql
      containers:
      - name: odoo-app
        image: asia-southeast1-docker.pkg.dev/odoo-lab-vsi/bidv-poc/bidv_poc
        ports:
        - containerPort: 8069
        env:
        - name: DB_PORT_5432_TCP_ADDR
          value: "localhost"
        - name: DB_PORT_5432_TCP_PORT
          value: "5432"
        - name: INSTANCE_HOST
          value: "127.0.0.1"
        - name: DB_PORT
          value: "5432"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: gke-cloud-sql-secrets
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: gke-cloud-sql-secrets
              key: password
        - name: DB_NAME
          value: "odoo"
      - name: cloud-sql-proxy
        # This uses the latest version of the Cloud SQL proxy
        # It is recommended to use a specific version for production environments.
        # See: https://github.com/GoogleCloudPlatform/cloudsql-proxy
        image: gcr.io/cloudsql-docker/gce-proxy:latest
        command:
          - "/cloud_sql_proxy"

          # If connecting from a VPC-native GKE cluster, you can use the
          # following flag to have the proxy connect over private IP
          - "-ip_address_types=PRIVATE"

          # tcp should be set to the port the proxy should listen on
          # and should match the DB_PORT value set above.
          # Defaults: MySQL: 3306, Postgres: 5432, SQLServer: 1433
          - "-instances=odoo-lab-vsi:asia-southeast1:odoo-bidv-doc=tcp:5432"
        securityContext:
          # The default Cloud SQL proxy image runs as the
          # "nonroot" user and group (uid: 65532) by default.
          runAsNonRoot: true
```

Trong file này thì cần chú ý là phần image các bạn có thể để image của Odoo trên Docker Hub, cái image này của mình là mình push lên **Artifact Registry** của GCP.

Chạy ```kubectl apply -f deployment.yaml``` để deploy. 

Cuối cùng ta tạo service để truy cập web từ bên ngoài với lệnh ```kubectl apply -f service.yaml```. File serivce thì như sau

```
apiVersion: v1
kind: Service
metadata:
  name: odoo-app
spec:
  type: LoadBalancer
  selector:
    app: odoo-app
  ports:
  - port: 80
    targetPort: 8069
```
