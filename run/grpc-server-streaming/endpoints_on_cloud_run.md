# Getting Started with cloud endpoints for cloud run with ESPv2

## Task List

1.  Create a project and deploy a sample backend gRPC service.
2.  Reserve a Cloud Run hostname for the ESPv2 Beta service.
3.  Create a gRPC API configuration document that describes your API, and
    configure the routes to your Cloud Run.
4.  Deploy the gRPC API configuration document to create a managed service.
5.  Build a new ESPv2 Beta Docker image with your Cloud Endpoints service
    configuration.
6.  Deploy the ESPv2 Beta container onto Cloud Run.
7.  Invoke a service.
8.  Track activity to your services.

To get set up:

1.  Create a new project. On the rest of this page, this project ID is referred
    to as **PROJECT_ID**
2.  Install gRPC and gRPC tools from the
    [gRPC go quickstart](https://grpc.io/docs/languages/go/quickstart/)

## Deploy the backend gRPC Cloud Run service

1.  Navigate to the directory of your Dockerfile then run:
    <pre>
    docker build -t gcr.io/<b>PROJECT_ID</b>/<b>IMAGE_NAME</b> .
    </pre>
2.  Use gcloud to authenticate docker
    <pre>gcloud auth configure-docker</pre>
3.  Push your image
    <pre>docker push gcr.io/<b>PROJECT_ID</b>/<b>IMAGE_NAME</b></pre>
4.  Deploy your image to Cloud Run
    <pre>
    gcloud run deploy <b>CLOUD_RUN_BACKEND_NAME</b>  --project=<b>PROJECT_ID</b> \ 
         --platform=managed  --region=<b>REGION</b> --image=gcr.io/<b>PROJECT_ID</b>/<b>IMAGE_NAME</b>
    </pre>

## Reserving a Clout Run hostname

1.  Set the region
    <pre>
    gcloud config set run/region <b>REGION</b>
    </pre>
2.  Deploy the sample image `gcr.io/cloudrun/hello` to Cloud Run. Replace
    <b>ESP_CLOUD_RUN_SERVICE_NAME</b> with the name that you want to use for the
    ESP service (for example: gateway).
    <pre>
    gcloud run deploy <b>ESP_CLOUD_RUN_SERVICE_NAME</b> \ 
    --image="gcr.io/cloudrun/hello" \ 
    --platform managed \ 
    --project=<b>PROJECT_ID</b>
    </pre>
    On successful completion, the command displays a message similar to the following:
    <pre>
    Service [<b>ESP_CLOUD_RUN_SERVICE_NAME</b>] revision [<b>ESP_CLOUD_RUN_SERVICE_NAME-REVISION_NUM</b>] has been deployed and is serving traffic at <b>ESP_CLOUD_RUN_SERVICE_URL</b>
    </pre>
    The <b>ESP_CLOUD_RUN_SERVICE_URL</b> is of the form `https://ESP_CLOUD_RUN_SERVICE_NAME-....a.run.app`\
    The <b>ESP_CLOUD_RUN_HOSTNAME</b> is the same as <b>ESP_CLOUD_RUN_SERVICE_URL</b> but without the `https://` prefix.

## Configuring Cloud Endpoints

1.  Create a descriptor file, `api_descriptor.pb` from your `.proto` file using `protoc`\
    Run the command:
    <pre>
    python3 -m grpc_tools.protoc \ 
    --include_imports \ 
    --include_source_info \ 
    --proto_path=. \ 
    --descriptor_set_out=api_descriptor.pb \ 
    api/v1/timeservice.proto
    </pre>
2.  Create a file called `api_config.yaml` in the same directory. Add the following contents to the file:
    <pre>
    # The configuration schema is defined by the service.proto file.
    # https://github.com/googleapis/googleapis/blob/master/google/api/service.proto
    
    type: google.api.Service
    config_version: 3
    name: <b>ESP_CLOUD_RUN_HOSTNAME</b>
    title: Cloud Endpoints + Cloud Run gRPC
    apis:
      - name: timeservice.TimeService
    usage:
      rules:
        # RouteChat methods can be called without an API Key.
        - selector: timeservice.TimeService.RouteChat
          allow_unregistered_calls: true
    backend:
      rules:
        - selector: "*"
          address: grpcs://<b>hostname_of_backend_grpc_cloud_run_service</b>
    </pre>
    To find the hostname of the backend service, open the Google Cloud console then navigate to you cloud run service. You'll find the hostname in the details tab under `url`.\
    ***Note:*** To use gRPC with TLS on Cloud Run, the address field must have the scheme `grpcs://` instead of `https://`\
    The value of the title ("Cloud Endpoints + Cloud Run gRPC") becomes the name of endpoint service.

## Deploying the Cloud Endpoints configuration

1.  Make sure you are in the directory where `api_descriptor.pb` and
    `api_config.yaml` are located.

2.  Run the command:
    <pre>
    gcloud endpoints services deploy api_descriptor.pb api_config.yaml
    </pre>

3.  When the deployment completes, a message similar to the following is displayed:
    <pre>
    Service Configuration [<b>CONFIG_ID</b>] uploaded for service ...
    </pre>
    Make note of <b>CONFIG_ID</b> as we'll use it later.

4.  Enable the following services if not already enabled:
    <pre>
    gcloud services enable servicemanagement.googleapis.com
    gcloud services enable servicecontrol.googleapis.com
    gcloud services enable endpoints.googleapis.com
    </pre>

5.  Enable the Cloud Endpoints service. To find the name of the endpoints service,
    go to the Endpoints page in the Cloud Console. The list of possible
    <b>ENDPOINTS_SERVICE_NAME</b> are shown under the Service name column.
    <pre>
    gcloud services enable <b>ENDPOINTS_SERVICE_NAME</b>
    </pre>

## Building a new ESPv2 Beta image

Build the Endpoints service config into a new ESPv2 Beta docker image. You will
later deploy this image onto the reserved Cloud Run service.

1.  Download this
    [script](https://github.com/GoogleCloudPlatform/esp-v2/tree/master/docker/serverless/gcloud_build_image)
    to your local machine.
2.  Make the script executable.
    <pre>
    chmod +x gcloud_build_image
    </pre>
3.  Run the script
    <pre>
    ./gcloud_build_image -s <b>ESP_CLOUD_RUN_HOSTNAME</b> \
    -c <b>CONFIG_ID</b> -p <b>PROJECT_ID</b>
    </pre>
4.  The script uses the gcloud command to download the service config, build the service config into a new ESPv2 Beta image, and upload the new image to your project container registry. The script automatically uses the latest release of ESPv2 Beta, denoted by the <b>ESP_VERSION</b> in the output image name. The output image is uploaded to:
    <pre>
    gcr.io/<b>PROJECT_ID</b>/endpoints-runtime-serverless:<b>ESP_VERSION</b>-<b>ESP_CLOUD_RUN_HOSTNAME</b>-<b>CONFIG_ID</b>
    </pre>

## Deploying the ESPv2 Beta container

1.  Deploy the ESPv2 Beta Cloud Run service with the new image you built above.
    <pre>
    gcloud run deploy <b>ESP_CLOUD_RUN_SERVICE_NAME</b> \
    --image="gcr.io/<b>PROJECT_ID</b>/endpoints-runtime-serverless:<b>ESP_VERSION</b>-<b>ESP_CLOUD_RUN_HOSTNAME</b>-<b>CONFIG_ID</b>" \
    --platform managed \
    --project=<b>PROJECT_ID</b>
    </pre>
    See [ESPv2 Beta flags](https://cloud.google.com/endpoints/docs/grpc/specify-esp-v2-startup-options#configuration_flags)
    for more info about different flags and startup options such as tls.

## Sending requests to the API

Send a bi-directional gRPC streaming request to the API:
<pre>
go run client/client.go -server <b>ENDPOINTS_SERVICE_NAME</b>:443 -auth-token $(gcloud auth print-identity-token) -chat
</pre>

## Tracking API activity

Look at the request logs for your API on the Logs Viewer page.
[View Endpoints request logs](https://console.cloud.google.com/logs/viewer?_ga=2.98032159.1097896375.1604335020-1049980475.1603729712)

## Creating a developer portal for the API

You can use Cloud Endpoints Portal to create a developer portal, a website that
you can use to interact with the sample API. \
To learn more, see
[Cloud Endpoints Portal Overview](https://cloud.google.com/endpoints/docs/grpc/dev-portal-overview).
