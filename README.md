# Microservice Production Readiness Checklist

These are the principles that I have developed over the years of my professional career or by adapting the available knowledge in the literature to my work. I like to use these principles before deploying a new microservice (application) to production. I consider it also a good starting point for a discussion.  

<em>(* Yes, a lot of the points here can be covered via sane defaults provided by service templating)</em>

## Table of Contents

1. [The Checklist](#the-checklist)
   - [General Rules](#general-rules)
   - [Documentation](#documentation)
   - [Testing and Quality](#testing-and-quality)
   - [Observability](#observability)
   - [Operations and Resiliency](#operations-and-resiliency)
   - [Database](#database)
   - [Security and Compliance](#security-and-compliance)
   - [Costs](#costs)
2. [References](#references)
3. [Bonus](#bonus)
   - [Independent Service Heuristics - Team Topologies](#independent-service-heuristics---team-topologies)

## The Checklist

### General Rules

- [ ] Write a **RFC** and publish it on architecture-guild channel if your change with the new microservice may affect the work of other teams
- [ ] **No shared database between different services** - a DB instance should only be used by one service exclusively.
- [ ] **Not breaking the one-hop rule** - [<em> “By default, a service should not call other services to respond to a request, except in exceptional circumstances.” A service should not call other services to respond to a request; it should be self-contained and manage its own data. Allowing a service to call on other services adds overhead to the request and can result in very slow or unresponsive service. </em>](https://thenewstack.io/are-your-microservices-overly-chatty/)  Also it leads to tightly coupled services and a more complex system causing cascading issues. If you see that you need multiple calls back and forth between several services to synthesize the response, you should consider merging these services into one.). The exception can be for example Backend For Frontend (like GraphQL) which can compose and aggregate data on top of other services.
- [ ] **Prefer APIs over Sharing SDKs**. Try to avoid using SDKs between the services, it is not needed. [<em>SDKs introduce another level of complexity, making the microservice team support not only API to their product but also the code that uses that API</em>](https://enterprisecraftsmanship.com/posts/how-to-build-microservices-wrong/). They are also introducing the problems like updating the version of SDK in multiple services. Also, it prevents asynchronous way of development between the teams. Because Team A needs to wait for Team B to finish the work around the SDK.
- [ ] If you can try to avoid making HTTP calls from Monolith to your service and the same in opposite direction: try to do not call Monolith, do not increase load there. Your service should be independent.
- [ ] Choose Boring Technology https://boringtechnology.club/ :)  

### Documentation

- [ ] **Readme.md** - self-explanatory service name, how to run it locally and domain/sub-domain, bounded context described
- [ ] **Architecture docs** / C4 Model diagrams
- [ ] **Service Catalog** integration (e.g. Backstage)
- [ ] **API Open Specification** file in root directory openapi.yaml
- [ ] API **versioning** if needed


### Testing and Quality

- [ ] **Linters** (with reports that can be exported to e.g. SonarQube)
- [ ] Automatic code **Formatter** (e.g. gofmt, ktfmt)
- [ ] **Test coverage** above 70% (use common sense, just getting to the required number of coverage is not a goal here)
- [ ] **Functional/e2e/acceptance** tests in place
- [ ] **Load Test**s (at least basic ones) especially if higher traffic is expected
- [ ] **Contract Tests** are recommended if there is service 2 service communication via HTTP (example: PACT tests)

### Observability

- [ ] **Logging**: All logs are written to STDOUT / STDERR. Logs are written in JSON. Configured verbosity levels. Correlation IDs, Mapped
Diagnostic Context (MDC) is used. https://12factor.net/logs. Do not log any sensitive data. Shipped to e.g. ELK, Stackdriver, etc.
- [ ] Integration with a **monitoring platforms**. Dashboards in place. (e.g. NewRelic / Prometheus / Grafana)
- [ ] **Monitoring dashboards** with Business Metrics (e.g. New Relic / Prometheus / Grafana)
- [ ] Integration with a **distributed tracing system**: (e.g. New Relic)
- [ ] Integration with **error tracking system** like (e.g. Sentry, New Relic) 
- [ ] **Alerts** are configured, there is Pager Duty integration in place and consider adding it to one of the escalation policies RunBooks (you may want to use Alert Manager integration)
- [ ] **Uptime**, domain, SSL monitoring (optional) (e.g. StatusCake)
- [ ] Real-time communication of application **status** (e.g. StatusPage)
- [ ] Measure latency using percentiles. Discover the expected peak latency of the system, when healthy. Measure it and know 99%ile of the response times. Discover the expected peak throughput of my system, when healthy. Measure requests-per-minute or requests-per-second

### Operations and Resiliency

- [ ] There is **staging environment** or any other **multi-tenancy environment** - Don’t push directly to production
- [ ] There is **autoscaling** in place (based on CPU, memory, traffic, events/messages e.g. HPA with K8S)
- [ ] **Graceful shutdown**: The application understands SIGTERM and other signals and will gracefully shutdown itself after processing the current task. https://12factor.net/disposability
- [ ] **Configuration** via environment: All important configuration options are read from the environment and the environment has higher priority over configuration files (but lower than the command line arguments). https://12factor.net/config
- [ ] **Health Checks**: Readiness and Liveness probes 
- [ ] **Resiliency**, dependencies failure check, connection/socket **timeouts**, **Retry Policies/circuit breakers** in place. Safe to retry - **idempotent** **calls** etc. (The timeout for external service should be equal to 99%ile of this dependency's latency when healthy plus some breathing room). You can consider using service mesh.
- [ ] **Resiliency**, If needed introduce the patterns like **rate limiting** and **backpressure** and/or **load shedding** to avoid overloading the system and making it unavailable for all users 
- [ ] **Resiliency**, implement **fallback logic** for every external dependency, for which this was possible and needed. Ask yourself “Is this resource needed to fulfill my business function, or can I (even partially) succeed without it?” (Remember that [Fallbacks can be tricky](https://nurkiewicz.com/2019/07/fallbacks-are-overrated-architecting.html) and implementing a **compensation job** might be required)
- [ ] **Resource limits**: Contains limits for memory, CPU, disk space, and any other available resources in the agreed format.
- [ ] **Artifact Managemen**t / Libraries & dependency proxy : JFROG Artifactory / Gitlab
- [ ] **Discovery / DNS** configured
- [ ] Setting **DNS Cache TTL** (hint for [Java](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/java-dg-jvm-ttl.html))
- [ ] **Dead letter queues** or/and resistance to "bad" messages (if queues are used)
- [ ] **Feature Flags** if needed (LaunchDarkly)
- [ ] **Locked versions of dependencies**: Dependencies for package managers are fixed, including minor versions (For example, cool_framework = 2.5.3). Committed lock files are also a good way to do this.
- [ ] Use private **Docker Registry** (AWS ECR , Gitlab)
- [ ] Use Immutable deployments and if possible Canary deployments
- [ ] When using containers like Docker - the only single process is running inside the container, with your application 
- [ ] Minimum 2 pods on production and/or PodDisruptionBudget/NodeAntiAffinity - whichever is appropriate to mitigate against a node failure - when running in Kubernetes
- [ ] Define [SLO/SLI/SLA](https://cloud.google.com/blog/products/devops-sre/sre-fundamentals-slis-slas-and-slos)
- [ ] Build applications with **Multi-tenancy** in mind (sites, regions, users, etc.)


### Database

- [ ] Follow best practices from Cloud Provider (e.g. [AWS Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.BestPractices.html)) and prepare for Fast Failover
- [ ] **Data Org support** (external team might have such requirements) - DB needs to be in provisioned mode instead of serverless if Bin Logs are used for exporting data, use bigger instances than t.small ones (e.g. t.small instances don’t support IAM access)
- [ ] **Database**’s Connection string and Connection pool configured for the needed workload
- [ ] **Database** (e.g. RDS - Aurora Writer/Reader endpoint consumed for better scalability, DocumentDB - use ReplicaSet in this case etc.)
- [ ] **Database** integration with Backup Tooling - test regularly your restore process
- [ ] Different **Database** users for RDS cluster admin and application usage
- [ ] Maintenance window defined
- [ ] Encryption enabled (e.g. with AWS KMS key)
- [ ] Data Migration, migration scripts etc. (e.g. Flyway, Liquibase, gh-ost, Percona) - [avoid manual changes on the database](https://softwareengineering.stackexchange.com/questions/207987/safely-fixing-production-database-data) (tip: use a separate process for migrations ) - executed as separate process/job

### Security and Compliance

- [ ] If your service does not absolutely need to be directly accessible from the public Internet, do not expose it publicly - hide it behind VPN (at least)
- [ ] If your service does need to be accessible through the public Internet
     - [ ] **Authentication/Authorization** in place if needed / JWT / Cognito / Auth0
     - [ ] Ensure it lives behind our Cloudfront **CDN** (and uses **WAF** if necessary)
- [ ] **Vulnerabilities scan check** (e.g. SonarQube)
- [ ] Run containers in [Rootless mode](https://docs.docker.com/engine/security/rootless/)
- [ ] **HTTPS** (if needed)
- [ ] Does not violate any licenses
- [ ] **GDPR** data not exposed (https://gdpr-info.eu/art-4-gdpr/)
- [ ] **PII data not logged or stored without any good reason** (ask your DPO) - [Best practices to avoid sending Personally Identifiable Information (PII)](https://support.google.com/adsense/answer/6156630?hl=en), Check Data Retention Policies
- [ ] Configure bot to upgrade dependencies (e.g. Renovate bot)

### Costs

- [ ] **Tags** for Infrastructure / Terraform resources
- [ ] **Costs Dashboard** for Weekly/Monthly costs (e.g. Cloudability)
- [ ] **Optimized CPU usage** (eg. [Reactive](https://spring.io/reactive) microservices) 
- [ ] Prepare **Rightsizing** plan 

### References

- https://srcco.de/posts/web-service-on-kubernetes-production-checklist-2019.html
- https://habr.com/en/post/438186/
- https://www.oreilly.com/library/view/production-ready-microservices/9781491965962/app01.html
- https://github.com/paunin/soa-checklist
- https://12factor.net
- [http://bit.ly/sredesign](http://bit.ly/sredesign) SRE checklist
- https://github.com/mercari/production-readiness-checklist
- https://docs.google.com/document/d/11HO1TNM8Pmawo8dHvyvTYB9uoPpB0ot1Jjs-WULYzGo/edit
- https://www.youtube.com/watch?v=zwpXhFcR4BA
- https://thenewstack.io/are-your-microservices-overly-chatty/
- [Jakub Nabrdalik - What I wish I knew when I started designing systems years ago](https://youtu.be/1HJJhGHC2A4) - https://jakubn.gitlab.io/wish-i-knew-architecture/
- https://fs.blog/2014/02/the-checklist-manifesto/
- Security checklist https://www.sensedeep.com/blog/posts/stories/web-developer-security-checklist.html
- https://grafana.com/blog/2021/10/13/how-were-building-a-production-readiness-review-process-at-grafana-labs/ - Checklist [here](https://docs.google.com/document/d/11HO1TNM8Pmawo8dHvyvTYB9uoPpB0ot1Jjs-WULYzGo/edit)
- SreConAsia19/PacificReliable by Design Adding Value in the Design Review Process - https://www.usenix.org/conference/srecon19asia/presentation/nolan 
> "Reviewing designs written by other engineers becomes an increasingly large (and important) part of our work life as we become more senior in our careers. We review designs for entire new systems written by partner developer teams. We review designs for pieces of automation to be developed and run by our own teams. Eventually we may find ourselves using review as a way to keep many teams in sync technically.

> Most of us however, don't have a systematic way to approach reviews. We read the proposal or attend the meeting, and we look to our experience to try and predict problems. This is valuable, and experience can't be replaced—but I believe we can do better by applying both our expertise and a checklist of things to consider for each design."

### Bonus

Because good design = Best Resilency.

#### Independent Service Heuristics - [Team Topologies](https://github.com/TeamTopologies/Independent-Service-Heuristics)

<em>Rules-of-thumb for identifying candidate value streams and domain boundaries by seeing if they could be run as a separate SaaS/cloud product.

- Could it make any logical sense to offer this thing "as a service"?
- Could you imagine this thing branded as a public cloud service (like AvocadoOnline.com )?
- Could this thing be managed a viable cloud service in terms of revenue and customers?
- Could the organisation currently track costs and investment in this thing separately from similar things?
- Could this thing operate with minimal data from other sources?
- Could this thing have a small/well-defined set of user types or customers (user personas)?
- Could a team or set of teams effectively build and operate a service based on this thing?</em>


