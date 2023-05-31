# Module 4 - Security Guardrails for Network-based Threats

Calico provides methods to enable fine-grained access controls between your microservices and external databases, cloud services, APIs, and other applications that are protected behind a firewall. You can enforce controls from within the cluster using DNS egress policies, from a firewall outside the cluster using the egress gateway. Controls are applied on a fine-grained, per-pod basis.

In this module, we will learn how to use Calico to create network policies to control access to and from a pod. For this, we will install two example application stacks: The Online Boutique and the Dev environment. Once the applications are deployed, we will create and test network security policies with different ingress and egress rules to demonstrate how the **workload access control** is done.

1. Installing the example application stacks:

   From the cloned directory, execute:
   ```
   kubectl apply -f pre
   ```

Included in the `pre` there folder are two applications that will be used in the exercises during the workshop. The diagram below shows how the elements of each application communicate between themselves.

![applications](https://github.com/tigera-solutions/cc-aks-detect-block-network-attacks/assets/104035488/842088b1-320c-4ada-8905-10d1dd02c57c)

There are also other objects that will be created for the workshop. We will lear about them later in the workshop.

   > **Note**: Wait until all the pods are up and running to move to the next step.

## Service Graph and Flow Visualizer

Connect to Calico Cloud GUI. From the menu select `Service Graph > Default`. Explore the options.

![service_graph](https://user-images.githubusercontent.com/104035488/192303379-efb43faa-1e71-41f2-9c54-c9b7f0538b34.gif)

Connect to Calico Cloud GUI. From the menu select `Service Graph > Flow Visualizations`. Explore the options.

![flow-visualization](https://user-images.githubusercontent.com/104035488/192358472-112c832f-2fd7-4294-b8cc-fec166a9b11e.gif)

## Security Policies

Calico Security Policies provide a richer set of policy capabilities than the native Kubernetes network policies, including:  

- Policies that can be applied to any kind of endpoint: pods/containers, VMs, and/or to host interfaces
- Policies that can define rules that apply to ingress, egress, or both
- Policy rules support:
  - Actions: allow, deny, log, pass
  - Source and destination match criteria:
    - Ports: numbered, ports in a range, and Kubernetes named ports
    - Protocols: TCP, UDP, ICMP, SCTP, UDPlite, ICMPv6, protocol numbers (1-255)
    - HTTP attributes (if using Istio service mesh)
    - ICMP attributes
    - IP version (IPv4, IPv6)
    - IP or CIDR
    - Endpoint selectors (using label expression to select pods, VMs, host interfaces, and/or network sets)
    - Namespace selectors
    - Service account selectors

### The Zero Trust approach

A global default deny policy ensures that unwanted traffic (ingress and egress) is denied by default. Pods without policy (or incorrect policy) are not allowed traffic until appropriate network policy is defined. Although the staging policy tool will help you find incorrect and missing policy, a global deny helps mitigate against other lateral malicious attacks.

By default, all traffic is allowed between the pods in a cluster. Let's start by testing connectivity between application components and across application stacks. All of these tests should succeed as there are no policies in place.

We recommend that you create a global default deny policy after you complete writing policy for the traffic that you want to allow. Use the stage policy feature to get your allowed traffic working as expected, then lock down the cluster to block unwanted traffic.

1. Create a staged global default deny policy. It will shows all the traffic that would be blocked if it were converted into a deny.

   - Go to the `Policies Board`
   - On the bottom of the tier box `default` click on `Add Policy`
     - In the `Create Policy` page enter the policy name: `default-deny`
     - On the `Applies To` session, click `Add Namespace Seletor`
       First, lets apply only to the `vote` namespace
       - Select Key... `kubernetes.io/metadata.name`
       - =
       - Select Value... `vote`
     - On the field `Type` select both checkboxes: Ingress and Egress.
     - You are done. Click `Stage` on the top-right of your page.

   The staged policy does not affect the traffic directly but allows you to view the policy impact if it were to be enforced. You can see the deny traffic in staged policy.

2. Based on the application design, the `db` lists on port `5432` and receive connections from the `worker` and the `result` microservices. 
   Let's use the Calico Cloud UI to create a policy to microsegment this traffic.

   - Go to the `Policies Board`
   - On the bottom of the tier box `platform` click on `Add Policy`
     - In the `Create Policy` page enter the policy name: `db`
     - Change the `Scope` from `Global` to `Namespace` and select the namespace `vote`
     - On the `Applies To` session, click `Add Label Seletor`
       - Select Key... `app`
       - =
       - Select Value... `db`
       - Click on `SAVE LABEL SELECTOR`
     - On the field `Type` select the checkbox for Ingress.
     - Click on `Add Ingress Rule`
       - On the `Create New Policy Rule` window,
         - Click on the `dropdown` with `Any Protocol`
         - Change the radio button to `Protocol is` and select `TCP`
         - In the field `To:` click in `Add Port` 
         - `Port is` 5432 - Save
         - In the field `From:`, click `Add Label Seletor`
           - Select Key... `app`
           - =
           - Select Value... `worker`
           - Click on `SAVE LABEL SELECTOR`
           `OR`, click `Add Label Seletor`
           - Select Key... `app`
           - =
           - Select Value... `result`
           - Click on `SAVE LABEL SELECTOR`
         - Click on the button `Save Rule`
     - You are done. Click `Enforce` on the top-right of your page.

3. Now, let's make use of the Recommend a Policy to create the policies for the other workloads.

   Let's start with the `redis` database.

   - Click in `Recommend a Policy`
   
     ![recommend a policy](https://github.com/tigera-solutions/cc-aks-detect-block-network-attacks/assets/104035488/49051f38-af1e-4786-91c4-490367fefcf4)
   
   - Select the `vote` namespace in the Namespace dropdown

     ![select namespace](https://github.com/tigera-solutions/cc-aks-detect-block-network-attacks/assets/104035488/8524b68e-2acc-4f7c-bb5f-8051b001317c)

   - Click in `Advanced Options` and select the `redis-xxxxxxx-*` from the dropdown
   
     ![workload](https://github.com/tigera-solutions/cc-aks-detect-block-network-attacks/assets/104035488/8a482686-c1ee-43ca-bcd0-05063862e8ff)
   
   - Note that the selector and the rules are already present!

     ![review the policy](https://github.com/tigera-solutions/cc-aks-detect-block-network-attacks/assets/104035488/94207043-dcc8-45df-a1e9-9f53171502d3)
   
   - Click on the Enforce button

     ![enforce](https://github.com/tigera-solutions/cc-aks-detect-block-network-attacks/assets/104035488/587fe7c7-d387-44fc-a722-f668b93a5e96)
   
   - By default, all the recommended policies are create in the `default` tier. You can drag and drop a policy to reorganize in the same tier or in another tier. Move the policy to the platform tier.

     ![move the policy](https://github.com/tigera-solutions/cc-aks-detect-block-network-attacks/assets/104035488/f48b066b-2339-451b-aae5-dd11b76702c7)

   Great! You just created a recommended policy. Easy, right? Now go ahead and create policies for all the other workloads in the vote namespace.

4. If you create all the policies correctly, at some point you will start seeing zero traffic being denied by your default-deny staged policy. At that point you can go ahead ane enforce the default-deny policy. Voilà! The vote namespace is now secure.

### About Tiers

Policies are processed in sequential order from top to bottom.

![policy-processing](https://user-images.githubusercontent.com/104035488/206433417-0d186664-1514-41cc-80d2-17ed0d20a2f4.png)

Two mechanisms drive how traffic is processed across tiered policies:

- Labels and selectors
- Policy action rules

For more information about tiers, please refer to the Calico Cloud documentation [Understanding policy tiers](https://docs.calicocloud.io/get-started/tutorials/policy-tiers)


--- 

[:arrow_right: Module 5 - Identity-aware Microsegmentation](/modules/module-5-identity-aware-microsegmentation.md)  <br>

[:arrow_left: Module 3 - Connect the AWS EKS cluster to Calico Cloud](/modules/module-3-connect-calicocloud.md)  
[:leftwards_arrow_with_hook: Back to Main](/README.md)  
