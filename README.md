## Installing MinIO on OpenShift 4.18 Using the MinIO Operator (Helm)

Follow these steps to install MinIO on OpenShift 4.18 using the MinIO Operator and Helm, based on the [official documentation](https://docs.min.io/community/minio-object-store/operations/deployments/k8s-deploy-operator-helm-on-kubernetes.html):

1. **Create a new project (namespace):**
   ```sh
   oc new-project minio-operator
   ```

2. **Add the MinIO Operator Helm repository:**
   ```sh
   helm repo add minio-operator https://operator.min.io/
   helm repo update
   ```

3. **Install the MinIO Operator using Helm:**
   ```sh
   helm install minio-operator minio-operator/operator \
     --namespace minio-operator \
     --create-namespace
   ```

4. **Verify the Operator is running:**
   ```sh
   oc get pods -n minio-operator
   ```

5. **Access the MinIO Operator Console:**
   - Expose the Operator Console service:
     ```sh
     oc expose service minio-operator-console -n minio-operator
     ```
   - Get the route URL:
     ```sh
     oc get route minio-operator-console -n minio-operator
     ```
   - Open the URL in your browser.

6. **Get the Operator Console credentials:**
   ```sh
   kubectl get secret console-sa-secret -n minio-operator -o jsonpath="{.data.CONSOLE_PASSWORD}" | base64 --decode
   ```
   - Username: `console`
   - Password: (output from the command above)

7. **Deploy a MinIO Tenant:**
   - You can deploy a tenant using the Operator Console UI, or by applying a YAML manifest. For example:
     ```sh
     kubectl apply -f https://raw.githubusercontent.com/minio/operator/master/examples/tenant.yaml -n minio-operator
     ```

8. **Expose the MinIO Tenant service (optional):**
   - After the tenant is created, expose its service as needed using `oc expose service ...`.

> **Note:** For production deployments, review and customize the tenant YAML and security settings as