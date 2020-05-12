
# EQ Performance Benchmark

This is a performance benchmarking tool designed to measure the performance of [EQ Survey Runner](https://github.com/ONSDigital/eq-survey-runner) using [locust](https://locust.io/).

This repository was heavily inspired by the [census performance tests](https://github.com/ONSdigital/census-eq-performance-tests).

## Installation

On MacOSX Catalina (10.15) if the Python packages fail to install there may be two versions of command line tools SDK present(`MacOSX10.15.sdk` and `MacOSX10.14.sdk`).
You need to remove 10.14 version by using: 

```
sudo rm -rf /Library/Developer/CommandLineTools/SDKs/MacOSX10.14.sdk
```

## Running a benchmark

The benchmark consumes a requests JSON file that contains a list of HTTP requests. This can either be created from scratch or generated from a HAR file. Example requests files can be found in the `requests` folder.

To run a benchmark, use:

```bash
pipenv run ./run.sh <REQUESTS_JSON> <HOST: Optional>
```
e.g.
```bash
pipenv run ./run.sh requests/test_checkbox.json
```

This will run 1 minute of locust requests with 1 user and no wait time between requests. The output files are `output_stats.csv`, `output_stats_history.csv` and `output_failures.csv`.

For the web interface:

```bash
REQUESTS_JSON=requests/test_checkbox.json HOST=http://localhost:5000 pipenv run locust
```

## Configuration

The following environment variables can be used to configure the locust test:

- `REQUESTS_JSON` - The filepath of the requests file relative to the base project directory
  - Defaults to `requests.json`
- `HOST` - The host name the benchmark is run against.
- `USER_WAIT_TIME_MIN_SECONDS` - The minimum delay between each user's GET requests
  - defaults to zero
- `USER_WAIT_TIME_MAX_SECONDS` - The maximum delay between each user's GET requests
  - defaults to zero

## Generating a requests file

Open the network inspector in Chrome or Firefox and ensure 'preserve log' is ticked. Run your manual or functional test. 

**Important:** The captured test should not include the `/session` endpoint.

After the test is complete, right-click on one of the requests in the network inspector and save the log as a HAR file. To generate a requests file from the HAR file run:

```bash
pipenv run python generate_requests.py <HAR_FILEPATH> <REQUESTS_FILEPATH> <SCHEMA_NAME>
```
e.g.
```bash
pipenv run python generate_requests.py requests.har requests/test_checkbox.json test_checkbox
```

## Dealing with repeating sections

Repeating sections utilise list_item_ids to differentiate between each repetition. However as these ids are generated at run time they are not available when a request file is generated by the above method. Without knowledge of those list_item_ids, GET requests which require them will naturally not be formatted correctly and will error.

To fix the issue we need to harvest the list_item_ids when they are available and then use those values to format future urls.

Our current implementation requires us to manually add in an element called "redirect_route" into the JSON request file in the first POST which we know will return the required list_item_id value.

For example in the census household survey we have a GET with the following

```
 {
    "method": "GET",
    "url": "/questionnaire/household/{person_1_list_id}/add-or-edit-primary-person/"
 }
```

We naturally need to know {person_1_list_id}, which can be obtained from the preceding POST. This would look similar to the following (remember dynamic list_item_id generation)

```
questionnaire/household/NZNuza/individual-interstitial/
```

Notice the "NZNuza", this is the list_item_id value and we can see its in position 3. To harvest this value we put in our "redirect_route" element and a path value matching the key to that position

```
"redirect_route": "/questionnaire/household/{person_1_list_id}/add-or-edit-primary-person/"

```

So putting it all together we would get

```
{
    "method": "POST",
    "url": "/questionnaire/primary-person-list-collector/",
    "data": {
      "you-live-here-answer": "Yes, I usually live here",
      "action[save_continue]": ""
    },
    "redirect_route": "/questionnaire/household/{person_1_list_id}/add-or-edit-primary-person/"
  },
  {
    "method": "GET",
    "url": "/questionnaire/household/{person_1_list_id}/add-or-edit-primary-person/"
  }
```

N.B Although the above example uses the same url in the redirect as the GET, it doesn't have to, it just looks cleaner, we only really care that the key ({person_1_list_id}) and the value returned from the POST align, the redirect_route value could be "/-/-/{person_1_list_id}/-/".

Manually adding in a redirect is only needed once for each list_item_id and you can do multiple at the same time.

---

## Deployment with [Helm](https://helm.sh/)

To deploy this application with helm, you must have a kubernetes cluster already running and be logged into the cluster.

Log in to the cluster using:

```
gcloud container clusters get-credentials survey-runner --region <region> --project <gcp_project_id>
```

You need to have Helm installed locally

1. Install Helm with `brew install kubernetes-helm` and then run `helm init --client-only`

2. Install Helm Tiller plugin for _Tillerless_ deploys `helm plugin install https://github.com/rimusz/helm-tiller`


### Deploying the app

To deploy the app to the cluster, run the following command:

```
helm tiller run \
-    helm upgrade --install \
-    runner-benchmark \
-    k8s/helm \
-    --set host=${HOST} \
-    --set container.image=${DOCKER_REGISTRY}/eq-survey-runner-benchmark:${IMAGE_TAG}
```

If you want to vary the default parameters Locust uses on start, you can specify them using the following variables:

- requestsJson - The filepath of the requests file relative to the base project directory
  - Defaults to `requests.json`
- locustOptions - The host name the benchmark is run against.
- userWaitTimeMinSeconds - The minimum delay between each user's GET requests
  - defaults to 1
- userWaitTimeMaxSeconds - The maximum delay between each user's GET requests
  - defaults to 2 
- output.bucket - Name of the GCS bucket in which the output should be stored.
- output.directory - Name of the directory within the GCS bucket in which the output should be stored.

e.g
```
helm tiller run \
-    helm upgrade --install \
-    runner-benchmark \
-    k8s/helm \
-    --set requestsJson=requests/census_individual_gb_eng.json \
-    --set locustOptions="--clients 1000 --hatch-rate 50 -L WARNING" \
-    --set host=https://your-runner.gcp.dev.eq.ons.digital \
-    --set container.image=eu.gcr.io/census-eq-ci/eq-survey-runner-benchmark:latest
```

## Visualising Benchmark Results

You can use the `visualise_results.py` script to visualise benchmark results over time. 

### Pre-Requisites

You need to be authenticated with GCP in order to download the benchmark results. To do this use:

`gcloud auth login --project <project_id>`

`gcloud auth application-default login`

Alternatively you can use a project service Account, you will need to make sure that the service account has permission to access the bucket [(see Google docs)](https://cloud.google.com/iam/docs/creating-managing-service-accounts).

Once the service account has been created you will need to download its JSON keys file [(see Google docs)](https://cloud.google.com/iam/docs/creating-managing-service-account-keys). The path for this file will be used as an environment variable for the `get_benchmark_results.py` script.

### Download Benchmark Results

If you are running the script using a service account you will need to set the path to the JSON key file as an evironment variable (see Pre-requisites above):
```bash
export GOOGLE_APPLICATION_CREDENTIALS="<path_to_json_credentials_file>"
```

To run the script and download results:
```bash
OUTPUT_BUCKET="<bucket_name>" pipenv run python -m scripts.get_benchmark_results
```

### Summarise the results
You can get a breakdown of the average response times for a result set by doing:
```bash
OUTPUT_DIR="outputs/daily-test" \
OUTPUT_DATE="2020-01-01" \
pipenv run python -m scripts.get_summary
```

This will output something like:
```
Questionnaire GETs average: 477ms
Questionnaire POSTs average: 573ms
All requests average: 524ms
```

If `OUTPUT_DATE` is not provided, then it will output a summary for all results within the provided directory.

### Run the Visualise Results script

The `visualise_results` script will run against any benchmark results stored in the directory that is passed as an environment variable to the script.
Optionally, you can also specify the number of days to visualise the results for. 

For example, to visualise results for the last 7 days:
```bash
OUTPUT_DIR="outputs/daily-test" \
NUMBER_OF_DAYS="7" \
pipenv run python -m scripts.visualise_results
```

A line chart will be generated and saved as `performance_graph.png`.


## Future Improvements

- Allow rerunning a test using the original timings rather than a random wait time between GET requests.
- Customisable HAR filters to allow this project to be used for other websites.
- Standardise method to saturate server or a benchmark format.
