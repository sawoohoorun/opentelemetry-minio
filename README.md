- [Installing MinIO on OpenShift 4.18 Using the MinIO Operator (Helm)](#installing-minio-on-openshift-418-using-the-minio-operator-helm)


## Installing MinIO on OpenShift 4.18 Using the MinIO Operator (Helm)

Follow these steps to install MinIO on OpenShift 4.18 using the MinIO Operator and Helm, based on the [official documentation](https://docs.min.io/community/minio-object-store/operations/deployments/k8s-deploy-operator-helm-on-kubernetes.html):

1. **Create a new project (namespace):**
   ```sh
   oc new-project minio-operator
   ```
4. **Grant the anyuid SCC to the MinIO Operator service account:**
   ```sh
   oc adm policy add-scc-to-user anyuid -n minio-operator -z minio-operator
   ```
   > This allows the MinIO Operator to run containers as any user ID, including UID 1000.


2. **Add the MinIO Operator Helm repository:**
   ```sh
   helm repo add minio-operator https://operator.min.io/
   helm repo update
   ```

2. **create custom values.yaml for openshift:**

   ```yaml
   operator:
     env:
       - name: OPERATOR_STS_ENABLED
         value: "on"
       - name: MINIO_CONSOLE_TLS_ENABLE
         value: "off"
     serviceAccountAnnotations: []
     additionalLabels: {}
     image:
       repository: quay.io/minio/operator
       tag: v5.0.17
       pullPolicy: IfNotPresent
     imagePullSecrets: [ ]
     runtimeClassName: ~
     initContainers: [ ]
     replicaCount: 2
     securityContext:
       runAsUser: 1000
       runAsGroup: 1000
       runAsNonRoot: true
       fsGroup: 1000
     containerSecurityContext:
       runAsUser: 1000
       runAsGroup: 1000
       runAsNonRoot: true
       allowPrivilegeEscalation: false
       capabilities:
         drop:
           - ALL
       seccompProfile: null
       #   type: RuntimeDefault
     volumes: [ ]
     volumeMounts: [ ]
     nodeSelector: { }
     priorityClassName: ""
     affinity:
       podAntiAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
           - labelSelector:
               matchExpressions:
                 - key: name
                   operator: In
                   values:
                     - minio-operator
             topologyKey: kubernetes.io/hostname
     tolerations: [ ]
     topologySpreadConstraints: [ ]
     resources:
       requests:
         cpu: 200m
         memory: 256Mi
         ephemeral-storage: 500Mi
   console:
     enabled: true
     additionalLabels: {}
     image:
       repository: quay.io/minio/operator
       tag: v5.0.17
       pullPolicy: IfNotPresent
     env: [ ]
     imagePullSecrets: [ ]
     runtimeClassName: ~
     initContainers: [ ]
     replicaCount: 1
     affinity:
       podAntiAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
           - labelSelector:
               matchExpressions:
                 - key: name
                   operator: In
                   values:
                     - minio-operator
             topologyKey: kubernetes.io/hostname
     tolerations: [ ]
     topologySpreadConstraints: [ ]
     resources:
       requests:
         cpu: 0.25
         memory: 512Mi
     securityContext:
       runAsUser: 1000
       runAsGroup: 1000
       runAsNonRoot: true
     containerSecurityContext:
       runAsUser: 1000
       runAsGroup: 1000
       runAsNonRoot: true
       allowPrivilegeEscalation: false
       capabilities:
         drop:
           - ALL
       seccompProfile: null
       #   type: RuntimeDefault
     readOnly: false


     ingress:
       enabled: false
       ingressClassName: ""
       labels: { }
       annotations: { }
       tls: [ ]
       host: console.local
       path: /
       pathType: Prefix
       number: 9090

     volumes:
       - name: tmp
         emptyDir: {}

     volumeMounts:
       - name: tmp
         readOnly: false
         mountPath: /tmp/certs/CAs

   ```

3. **Install the MinIO Operator using Helm:**
   ```sh
   helm install \
   --namespace minio-operator \
   --create-namespace \
   operator minio-operator/operator \
   -f minio-operator-values.yaml
   ```

4. **Grant the anyuid SCC to the MinIO Operator service account:**
   ```sh
   oc adm policy add-scc-to-user anyuid -n minio-operator -z minio-operator
   ```
   > This allows the MinIO Operator to run containers as any user ID, including UID 1000.


5. **Verify the Operator is running:**
   ```sh
   oc get pods -n minio-operator
   ```

6. **Access the MinIO Operator Console:**
   - Expose the Operator Console service:
     ```sh
     oc expose service minio-operator-console -n minio-operator
     ```
   - Get the route URL:
     ```sh
     oc get route minio-operator-console -n minio-operator
     ```
   - Open the URL in your browser.

7. **Get the Operator Console credentials:**
   ```sh
   oc get secret console-sa-secret -n minio-operator -o jsonpath="{.data.CONSOLE_PASSWORD}" | base64 --decode
   ```
   - Username: `console`
   - Password: (output from the command above)

8. **Deploy a MinIO Tenant:**
   - You can deploy a tenant using the Operator Console UI, or by applying a YAML manifest. For example:
     ```sh
     kubectl apply -f https://raw.githubusercontent.com/minio/operator/master/examples/tenant.yaml -n minio-operator
     ```

9. **Expose the MinIO Tenant service (optional):**
   - After the tenant is created, expose its service as needed using `oc expose service ...`.

> **Note:** For production deployments, review and customize the tenant YAML and security settings as- [Installing MinIO on OpenShift 4.18 Using the MinIO Operator (Helm)](#installing-minio-on-openshift-418-using-the-minio-operator-helm)
