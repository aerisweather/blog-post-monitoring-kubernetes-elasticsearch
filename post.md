# Monitoring Kubernetes with Elasticsearch, Kibana, and Heapster

We recently took the plunge into using Kubernetes (https://kubernetes.io/) to orchestrate containers for a number of our web services and data ingest pipelines. While we have been generally pleased with the experience, metrics, monitoring, and logging has continued to be a major pain point for us. With no out-of-the-box system available to centralized logging, we've resorted to scanning log files on individual pods, with a lot of guess-and-check work in matching server logs to application behavior. And without any human-readable interface for viewing application metrics, it's really hard to construct a story describing why the application behaves the way it does. 

It was clear that we needed a better solution, one that could aggregate log data across many different services, and serve the data in a way that tells a meaningful story. So we started with making a list of things we wanted to know:

* Metrics per pod (CPU, memory usage, latency, application errors), so we can identify and correct misbehaving pods
* Scheduling behavior - how are pods being scheduled across our available nodes
* Resource allocation vs actual usage, so we can right-size our resource limits

Some of this information is available with the kubernetes dashboard addon (https://github.com/kubernetes/dashboard), but data is only available for a short period of time, and the dashboard UI doesn't provide the level of customization we need to get real insight into our application behavior.

## The Elastic Stack

After spending some time looking into available monitoring solutions, it quickly became clear that some version of the Elastic Stack (https://www.elastic.co/products) is the way to go. It's very well documented with broad community support, and an advanced feature set. Plus pricing for the hosted service is very reasonable (https://www.elastic.co/cloud/as-a-service/pricing), which means I don't have to worry about maintaing the stack itself, for the time being.

The only downside was that we had no idea how to get kubernetes metrics into Elasticsearch.

## How the heck do we ingest kubernetes metrics?

I was super overwhelmed trying to answer this question. There are so many different tools that were unfamiliar to me -- Elasticsearch, Kibana, Logstash, Beats, cAdvisor, Fluentd, Heapster, Kubernetes internals -- I didn't know where to start.

Finally, I ran into some documentation about an Elasticsearch "sink" for Heapster (https://github.com/kubernetes/heapster/blob/master/docs/sink-configuration.md#elasticsearch). This was the magic bullet I needed to get started.

## Ingesting Heapster metrics into Elasticsearch

If you're unfamiliar with Heapster (as I was), it is a tool that collects metrics from kubernetes, and ingests them into any number for backends ("sinks").

You may already have heapster running on your kubernetes cluster. An easy way to check is to run:

```bash
$ kubectl get pods --all-namespaces | grep heapster
```

If you see some output, it's already installed. If not, you can install it with an addon maintained by kops (https://github.com/kubernetes/kops/blob/master/docs/addons.md#monitoring-with-heapster---standalone):

```bash
$ kubectl create -f https://raw.githubusercontent.com/kubernetes/kops/master/addons/monitoring-standalone/v1.6.0.yaml
```

All we need to do now is configure the Elasicsearch sink. To do this we need to modify the Heapster deployment:

```bash
$ kubectl edit deployment heapster --namespace=kube-system
```

This will open up an editor, with the heapster deployment configuration.

Find a line that looks like:

```yaml
spec:
  containers:
  - command:
    - /heapster
    - --source=kubernetes.summary_api:''
```

Assuming you have an Elasticsearch server up an running, all you need to do is add a flag for the new sink, like so:

```yaml
spec:
  containers:
  - command:
    - /heapster
    - --source=kubernetes.summary_api:''
    # Configure the ES sink
    # You'll need to use your own credentials and DB hostname
    - --sink=elasticsearch:?nodes=https://[DB_HOST]:9243&esUserName=[DB_USER]&esUserSecret=[DB_PASS]&sniff=false&maxRetries=3
```

This will trigger an update of the `heapster` deployment. You may want to watch the heapster logs for a few minutes, to make sure everything's working:

```bash
$ kubectl get pods --all-namespaces | grep heapster
kube-system         heapster-3327791745-qhwgf                             2/2       Running   0          5d
$ kubectl logs -f heapster-3327791745-qhwgf
```

Now we just need to create an index pattern in Kibana:
(images/heapster-cpu-index-pattern.png)

And we should start seeing logs showing up in the `Discover` view:
(images/heapster-cpu-discover.png)

### Massaging the Heapster Data

Heapster gives us a lot of really useful information out of the box (https://github.com/kubernetes/heapster/blob/master/docs/storage-schema.md). However, there are a few tweaks we're going to want to make to the data structure, to make it easier to work with:

* Heapster uses separate timestamp fields for each metric type, which makes it difficult to visualize the data with Kibana. We'll merge the fields into a single `Timestamp` field.
* Heapster logs kubernetes labels as a single string, which makes it difficult to run queries/filter against label values. We'll split the `labels` field up into key-value fields.
* Heapster logs memory in bytes and CPU usage in millicores. Nothing wrong with that, but sometimes it's nice to work with larger units. We'll add fields for memory in Gibibytes, and CPU in cores.

How are we going to do all this? Ingest pipelines (https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html) to the rescue!

Take a look at the Ingest Processors documentation (https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest-processors.html), or if you're lazy, you can use the pipeline I've prepared (resources/heapster-pipelines.json).

You can create the ingest pipeline from the Dev Tools console in Kibana. Just name your pipeline `heapster`, by hitting the `PUT _ingest/pipeline/heapster` endpoint, with the heapster-pipeline.json document (resources/heapster-pipelines.json) as payload:

(images/heapster-pipeline.png)

Finally, we need to configure Heapster's Elasticsearch sink to use our pipeline.

Edit the `heapster` kubernetes deployment like we did before, and add a `pipeline=heapster` query parameter to the end of the `--sink` config:

```yaml
spec:
  containers:
  - command:
    - /heapster
    - --source=kubernetes.summary_api:''
    - --sink=elasticsearch:?nodes=https://[DB_HOST]:9243&esUserName=[DB_USER]&esUserSecret=[DB_PASS]&sniff=false&maxRetries=3&pipeline=heapster
```

Heapster will restart, and all new data will be transformed via our new ingest pipeline.

### Mapping Heapster Data

By default, Elasticsearch creates data-type mappings dynamically ()https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-mapping.html). I learned the hard way that this can be more of a headache than a time saver. If the first report of the day has your cpu usage at an even 3.0 cores, Elasticsearch will call that field a `long`. Any floating-point values ingested after that will result in mapping conflicts.

Mapping conflicts make that field unavaible for queries. A sesolving mapping conflicts is a real pain, so we're going to setup our mappings right off the bat.

Lucky for you, I created a mapping template (https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-templates.html) for you (resources/heapster-mapping-template.json). If you want to create your own, just run `GET /heapster*/_mapping` to see the existing dynamically-created mapping, then tweak it to your needs.

Once you have your template ready, send it as payload to `PUT /_template/heapster-mappings`

(images/heapster-mappings-template.png)

All new data from heapster will use this template. To prevent any mapping conflicts with the old data, you'll want to either reindex (https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html) or just delete any old indexes:

```
DELETE /heapster*
```

## Application Logs and Metrics
 
I won't go too much into application logging, as this will be specific to your use case and environment. But I will say that we decided to send log data directly to Elaticsearch using Winston (https://github.com/vanthome/winston-elasticsearch) (a Node.js logger), rather than running Logstash or Fluentd to collect container logs. It just seemed a lot more straightforward for us.

If you do want to try collecting logs from your containers' stdout, running a Fluentd DaemonSet seems like the way to go (https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/fluentd-elasticsearch/fluentd-es-ds.yaml)

## Visualizing with Kibana

It took a bit of playing around, but we've managed to create some nice dashboards in Kibana. 

For example, here are some graphs showing node CPU/memory usage vs kubernetes resource requests. You can see how pods were rescheduled during a recent deployment.

(images/kibana-node-allocation.png)

Because we parsed out the kubernetes `labels` log value, we can also get CPU/memory usage graphs split by service type:

(images/allocation-by-service.png)

And here we have a list of application error logs, right next to a chart of errors split by serviceId and error category.  

(images/error-logs.png) 

# Conclusions

With all of this data in Elasticsearch and Kibana, it makes it much easier to tell actionable stories about how our systems are behaving. So far, this information has allowed us to

- Identify a number of bugs in our system, and drastically reduce the error response rates of our Aeris Maps Platform API (https://www.aerisweather.com/develop/maps/)
- Tune our kubernetes resource limits to better match actual usage, allowing us to scale down servers that weren't being fully utilized
- Easily conduct A/B performance tests against new features, leading to major performance improvements.

It took a bit of effort to get going, but the payoff was certainly worth the investment.