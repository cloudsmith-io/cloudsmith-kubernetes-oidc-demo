# Cloudsmith OIDC Token Demo

This project demonstrates how to use Kubernete's service account tokens to populate a Kubernetes image pull secret for Cloudsmith. It consists of two main components:

1. A CronJob (`job.yaml`) that regularly fetches a Cloudsmith token using the Kubernetes service account token.
2. A demo deployment (`demo.yaml`) that uses the image pull secret created by the CronJob to fetch a Cloudsmith image.

Note: This is a proof of concept and should be adapted for production use. Adjust the implementation to fit your specific environment and adhere to your organisation's security practices.

## Assumptions

This demo assumes you are running kubernetes via docker on desktop on a mac and have access to ngrok

## Setup Instructions

1. Create a temporary directory to work in:
   ```bash
   mkdir -p /tmp/cloudsmith-oidc-token-demo
   cd /tmp/cloudsmith-oidc-token-demo
   ```

2. Start the web server and ngrok:
   ```bash
   python -m http.server 8000
   ngrok http 8000
   ```

3. Fetch and modify the OpenID configuration:
   ```bash
   mkdir -p .well-known
   kubectl get --raw /.well-known/openid-configuration > .well-known/openid-configuration
   kubectl get --raw /openid/v1/jwks > jwks.json
   ```
   Update `.well-known/openid-configuration`:
   ```json
   {
     "issuer": "https://YOUR_NGROK_SUBDOMAIN.ngrok.app",
     "jwks_uri": "https://YOUR_NGROK_SUBDOMAIN.ngrok.app/jwks.json",
     ...
   }
   ```

4. Create an openid configuration on cloudsmith providing your https://YOUR_NGROK_SUBDOMAIN.ngrok.app as the URL and binding it to a service account of your choice.

5. Modify the Kubernetes API server:
   ```bash
   docker run -it --rm --privileged --pid=host justincormack/nsenter1
   vi /etc/kubernetes/manifests/kube-apiserver.yaml
   ```
   Update:
   ```yaml
   - --service-account-issuer=https://YOUR_NGROK_SUBDOMAIN.ngrok.app
   ```

6. Apply configurations:
   ```bash
   kubectl apply -f job.yaml
   kubectl apply -f demo.yaml
   ```
   Note: Update `demo.yaml` with your Cloudsmith organisation and repository details.

7. Manually trigger the CronJob to run immediately:
   ```bash
   kubectl create job --from=cronjob/cloudsmith-secret-creator cloudsmith-secret-creator-manual
   ```
   This command creates a one-time Job from the CronJob, allowing you to verify the setup without waiting for the scheduled run.

## Pushing an Example Image

To push an example image to your Cloudsmith repository:

```bash
docker pull ubuntu:latest
docker tag ubuntu:latest docker.cloudsmith.io/YOUR_ORG/YOUR_REPO/ubuntu:latest
docker push docker.cloudsmith.io/YOUR_ORG/YOUR_REPO/ubuntu:latest
```

Replace `YOUR_ORG` and `YOUR_REPO` with your Cloudsmith organisation and repository names.

## Verification

To verify that everything is working correctly:

1. Check that the CronJob has run successfully:
   ```bash
   kubectl get cronjobs
   kubectl get jobs
   ```

2. Verify that the secret has been created:
   ```bash
   kubectl get secrets cloudsmith-registry-secret
   ```

3. Check the status of the demo deployment:
   ```bash
   kubectl get deployments
   kubectl get pods
   ```

If the pod is running, it means the image was successfully pulled using the secret created by the CronJob.

## Troubleshooting

- If the CronJob fails, check the logs of the job's pod:
  ```bash
  kubectl get pods
  kubectl logs <pod-name>
  ```

- If the demo deployment fails to pull the image, check that the secret is correctly formatted and contains the necessary data:
  ```bash
  kubectl get secret cloudsmith-registry-secret -o yaml
  ```

- Ensure that the Kubernetes API server has restarted successfully after modifying the configuration. You can check the API server pod status:
  ```bash
  kubectl get pods -n kube-system | grep api-server
  ```

- If you encounter issues after updating the `--service-account-issuer` flag, you may need to restart the API server manually. You can do this by deleting the API server pod in the `kube-system` namespace:
  ```bash
  kubectl delete pod -n kube-system -l component=kube-apiserver
  ```
  Kubernetes will automatically create a new API server pod with the updated configuration.
