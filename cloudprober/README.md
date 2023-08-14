# Wasmer Edge Monitoring with CloudProber

In Wasmer's SRE hiring Task, I was assigned to monitor the functionality of targets and run different checks against those targets.
Per task's request, I used Cloudprober to monitor Wasmer's edge nodes and assess the functionality of the following:

  - DNS
  - HTTP/HTTPS
  - PING (ICMP)

Cloudprober offers a dependable and user-friendly solution for monitoring edge node availability and performance.
Through an "active" monitoring approach, Cloudprober conducts probes on or against these systems to validate their optimal operation.



The task involved monitoring nodes equipped with both IPV4 and IPV6 addresses using Cloudprober.
Cloudprober employs a modular monitoring approach through the use of "probes."
A probe carries out specific actions, often directed at a group of targets, to validate the systems' alignment with consumer expectations.
For instance, an HTTP probe initiates an HTTP request to a web server, confirming its availability. These Cloudprober probes execute at regular intervals and report the outcomes as a collection of metrics.

The task's description indicated the necessity for seamless deployability using the official Helm chart. So, I refrained from altering the official Helm chart, opting instead to consolidate the probe composition within a single file.

The creation process was managed through Cloudprober's Go Template feature, utilizing macros provided by the Cloudprober team.

As mentioned above, we'll be monitoring "PING", "HTTP" and "DNS" of the following addresses:

  - IPV4: 5.78.72.41, 5.78.97.58, 5.161.216.132, 5.161.113.199, 5.161.212.179, 5.161.213.176, 5.161.90.11, 88.99.141.253, 88.99.240.24, 178.249.213.111, 51.161.198.52
  - IPV6: 2a01:4ff:1f0:d1a9::1, 2a01:4ff:1f0:8a73::1, 2a01:4ff:f0:c963::1, 2a01:4ff:f0:a1d8::1, 2a01:4ff:f0:d424::1, 2a01:4ff:f0:41c4::1, 2a01:4ff:f0:8ce2::1, 2a01:4f8:10a:2606::2, 2a01:4f8:10b:6f::2, null, fe80::d250:99ff:fedd:5d49::1



Also, we need to ensure that the following domains are reachable using both HTTP (port 80) and HTTPS (port 443), over both IPv4 and IPv6. These checks should also make sure that the HTML response looks somewhat as expected:

  - wasix.org
  - docs.wasmer.io
  - wasmer-docs.wasmer.app
  - wasmer.sh

Every Edge server operates a DNS server, and it's important to verify that the DNS service on each server is functional. This service should be able to provide IP addresses for the domain names included in the DNS checks.


#### Config Variables Structure
The initial idea that popped up was to organize the provided IPv4 and IPv6 addresses into two groups, sort of like creating lists, and then proceed with the task.
In the world of Go's templating, there's no direct method to set up a list of values, but luckily, Cloudprober's team has our backs with [custom macros](https://github.com/cloudprober/cloudprober/blob/master/config/config.go), which really simplify the whole process of setting up configurations!

We'll be using 2 macros:

  ```
  mkSlice
  mkMap
  ```

The list of IP's, list of domains and the list of type checks will each be stored in their own slice, and the overall configurations will be stored in a map. Here's how the variables will look like:

```go
{{ $domains :=  mkSlice "wasix.org" "docs.wasmer.io" "wasmer-docs.wasmer.app" "wasmer.sh" }}
{{ $checkTypes := mkSlice "PING" "HTTP" "DNS" }}
{{ $ipv4 := mkSlice "51.161.198.52" "178.249.213.111" "88.99.240.24" "88.99.141.253" "5.161.90.11" "5.161.213.176" "5.161.212.179" "5.161.113.199" "5.161.216.132" "5.78.97.58" "5.78.72.41" }}
{{ $ipv6 := mkSlice "2a01:4ff:1f0:d1a9::1" "2a01:4ff:1f0:8a73::1" "2a01:4ff:f0:c963::1" "2a01:4ff:f0:a1d8::1" "2a01:4ff:f0:d424::1" "2a01:4ff:f0:41c4::1" "2a01:4ff:f0:8ce2::1" "2a01:4f8:10a:2606::2" "2a01:4f8:10b:6f::2" }}

{{ template "IPDic" mkMap "ipver" "IPV4" "domains" $domains "checkTypes" $checkTypes "AllIPS" $ipv4 }}
{{ template "IPDic" mkMap "ipver" "IPV6" "domains" $domains "checkTypes" $checkTypes "AllIPS" $ipv6 }}
```


 ##### PING:
First, let's check that all the servers are alive and reachable, both over IPv4 and IPv6.
We'll use Cloudprober's "PING" probe to achieve this.
Like mentioned before, I refrained from altering the Cloudprobers's official Helm chart, and instead I opted to use the Go Template language to achieve the desired state.

For `PING` we won't need the domains, so we'll range over the IPs and create a `PING` probe for each IP:

```go
{{ with $ipTargets := $allIPs }}
        {{ range $_, $ip := $ipTargets }}

          {{ if eq $checkType "PING" }}
            probe {
              name: "{{$checkType}}_{{$ipVersion}}_{{$ip}}"
              type: {{$checkType}}
              ip_version: {{$ipVersion}}
              targets {
                host_names: "{{$ip}}"
              }
            }
            // REST OF THE CODE...
```

This is enough for creating PING probes for all the IPs provided, both V4 and V6 (notice that we're also providing the `ip_version` as well)

##### HTTP:

Next, let's take a look at the `HTTP` probe.
In the task description, it was mentioned that the list of provided domains should be reachable through both HTTP (port 80) and HTTPS (port 443), and over both IPv4 an IPv6. The checks should also make sure that the html response looks roughly as expected. 

Let's first take a step back and look at the bigger picture. 
Say you want to force a *resolve* for a domain so your local DNS resolver won't resolve the domain's IP address and you'd force the domain to use a specific IP Address.
In Curl, it's easy enough, we can run the following command:

```
$ curl --resolve "wasix.org:80:51.161.198.52" -I http://wasix.org
HTTP/1.1 200 OK
Content-Type: text/html
Accept-Ranges: bytes
Last-Modified: Thu, 01 Jan 1970 00:00:00 GMT
Cache-Control: public, max-age=86400
Date: Mon, 14 Aug 2023 09:48:30 GMT
Content-Length: 147170
X-Edge-App-Version-Id: dav_dn73IatguyJn
X-Wasmer-Request-Id: fe75d7ef-892b-4fe3-b83f-bcd6f449ed19

```

As you can see, we're forcing curl to use the IP we provided for sending a request to the domain.
Now, obviously, we don't have the `--resolve` flag provided to us in Cloudprober, but let's not lose hope! 
There's another trick for achieving the same results with Curl. We can you the Host header to force a request to a domain be resolved with our provided IP. Here's the equivillant request in curl using the host header:

```
$ curl -H "Host: wasix.org" http://51.161.198.52 -I
HTTP/1.1 200 OK
Content-Type: text/html
Accept-Ranges: bytes
Last-Modified: Thu, 01 Jan 1970 00:00:00 GMT
Cache-Control: public, max-age=86400
Date: Mon, 14 Aug 2023 09:52:01 GMT
Content-Length: 147170
X-Edge-App-Version-Id: dav_dn73IatguyJn
X-Wasmer-Request-Id: 7094954b-c67f-49eb-bbc4-d2c613b8f1e5
```

Unfortunately, this won't work in `HTTPS`. Because, we're not providing the server name in the SNI.
Fortunately, our friends at Cloudprober provided us with this configuration key in the `tlsConfig` directive to override the `serverName`.

Here's how we're handling the `HTTP` probe in our config:

```go
{{ with $domainTargets := $allDomains }}
              {{ range $_, $domain := $domainTargets }}

                {{ if eq $checkType "HTTP" }}
                  {{ with $httpCheckTypesTarget := mkSlice "HTTP" "HTTPS" }}
                    {{ range $_, $httpCheckType := $httpCheckTypesTarget }}

                      probe {
                        name: "{{$httpCheckType}}_{{$ip}}_{{$domain}}"
                        type: {{$checkType}}
                        ip_version: {{$ipVersion}}
                        http_probe {
                          protocol: {{$httpCheckType}}
                          headers: {
                            name: "Host"
                            value: "{{$domain}}"
                          }
                          {{ if eq $httpCheckType  "HTTPS" }}
                            tls_config {
                              server_name: "{{$domain}}"
                            }
                          {{ end }}
                        }
                        targets {
                          host_names: "{{$ip}}"
                        }
                        validator {
                          name: "status_code_2xx"
                          http_validator {
                            success_status_codes: "200-299"
                          }
                        }
                        validator {
                          name: "HTML_re"
                          regex: "<[^>]+>"
                        }
                      }

                    {{ end }}
                  {{ end }}
```

Notice that now we're also ranging over the domains slice. We're also using 2 validators.
We're checking if the status code is between 200 and 299 and if the response HTML looks as we expect it to look like.


Next, DNS!

##### DNS
For DNS, we're not doing anything special. Just like `HTTP`, we're ranging over the domains as well, and teseting the functionality of the DNS servers per IP/domain. 
We're also using the `min_answers` directive to check the minimum answers in the answer we get from the DNS request, which is 2.
Because we're checking with both IPv4 and IPv6, we're also checking `A` and `AAAA` records of those IPs/Domains. here's how the template would look like:

```go
{{ else if eq $checkType "DNS" }}
                  {{ with $dnsCheckTypesTarget := mkSlice "A" "AAAA" }}
                    {{ range $_, $dnsCheckType := $dnsCheckTypesTarget }}

                      probe {
                        name: "{{$checkType}}_{{$dnsCheckType}}_{{$ip}}_{{$domain}}"
                        type: {{$checkType}}
                        ip_version: {{$ipVersion}}
                        dns_probe {
                          resolved_domain: "{{$domain}}"
                          query_type: {{$dnsCheckType}}
                          min_answers: 2
                        }
                        targets {
                          host_names: "{{$ip}}"
                        }
                      }

                    {{ end }}
                  {{ end }}
                {{ end }}
```


And that's about it! We're also adding the Prometheus surfacer to the end of the template.
Now, if we run the following command, we'll have Cloudprober up and running, monitoring exactly what we need it to monitor!

Install the Cloudprober officail Helm chart:
```
helm repo add cloudprober https://helm.cloudprober.org/
helm repo update
helm install cloudprober cloudprober/cloudprober -n cloudprober --create-namespace
```
You can either add config directly in the values.yaml, or you can provide config on the command line through the --set-file flag.
```
helm upgrade --install cloudprober cloudprober/cloudprober \
    --set-file config=<PATH_TO_THE_TEMPLATE_FILE> -n cloudprober
```
