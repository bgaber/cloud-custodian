[Cloud Custodian Documentation — Cloud Custodian  documentation](https://cloudcustodian.io/docs/index.html)

[Cloud Custodian](https://cloudcustodian.io/)

[cloud-custodian/cloud-custodian](https://gitter.im/cloud-custodian/cloud-custodian)

[Cloud Custodian — Overview and deployment of cloud governance](https://medium.com/manomano-tech/cloud-custodian-overview-and-deployment-of-cloud-governance-d8e468fb4ab4)

Cloud Custodian is a tool that unifies the dozens of tools and scripts most organizations use for managing their public cloud accounts into one open source tool. It uses a stateless rules engine for policy definition and enforcement, with metrics, structured outputs and detailed reporting for clouds infrastructure. It integrates tightly with serverless runtimes to provide real time remediation/response with low operational overhead.

Organizations can use Custodian to manage their cloud environments by ensuring compliance to security policies, tag policies, garbage collection of unused resources, and cost management from a single tool.

Cloud Custodian can be bound to serverless event streams across multiple cloud providers that maps to security, operations, and governance use cases. Custodian adheres to a compliance as code principle, so you can validate, dry-run, and review changes to your policies.

Cloud Custodian policies are expressed in YAML and include the following:

- The type of resource to run the policy against

- Filters to narrow down the set of resources

- Actions to take on the filtered set of resources

The CompuCom SRE team has implemented Cloud Custodian on two platforms.  The first implementation, which only provides e-mail notifications has its accounts.yml and policies.yml files located in the BitBucket infra-report-ccustodian repository and is run by a scheduled Drone CI process: (https://drone.sre.compucom.com/compucomtechservices/infra-report-ccustodian).  The second implementation, which is for adhoc executions, is installed on SPLAWSSREDKR01.  Following are instructions on how to run an adhoc Cloud Custodian execution on SPLAWSSREDKR01:

`sudo su -`

`cd /usr/local/bin`

`source c7n-org/bin/activate`

`c7n-org run -c /home/awsadmin/bg216063/cloud-custodian/accounts.yaml -s /tmp -u /home/awsadmin/bg216063/cloud-custodian/policy.yaml`

The policy is validated automatically when you run it, but you can also validate it separately:

`custodian validate /home/awsadmin/bg216063/cloud-custodian/policy.yaml`

You can also check which resources are identified by the policy, without running any actions on the resources by using the --dryrun option shown below:

`c7n-org run -c /home/awsadmin/bg216063/cloud-custodian/accounts.yaml -s /tmp -u /home/awsadmin/bg216063/cloud-custodian/policy.yaml --dryrun`

In the execution above the **accounts.yaml** and **policy.yaml** are located in my directory (/home/awsadmin/bg216063/cloud-custodian).  You can look there for many examples of policy files and then create policy files in your own directory.  Regardless of the actions specified in the policy file, there will be a **resources.json** file containing the output of every run located in this directory:

**/tmp/{account}/{region}/{name from policy file}, eg. /tmp/us-prod/us-east-1/mcafee-security-groups**

Our accounts.yaml file contains this content:

```
accounts:
    - account_id: '472510080448'
      name: us-shared
      regions:
      - us-east-1
      role: arn:aws:iam::472510080448:role/sre-ec2-role-assumed
    - account_id: '995437807815'
      name: us-dev
      regions:
      - us-east-1
      role: arn:aws:iam::995437807815:role/sre-ec2-role-assumed
    - account_id: '300899438431'
      name: us-test
      regions:
      - us-east-1
      role: arn:aws:iam::300899438431:role/sre-ec2-role-assumed
    - account_id: '122639376858'
      name: us-prod
      regions:
      - us-east-1
      role: arn:aws:iam::122639376858:role/sre-ec2-role-assumed
    - account_id: '172507017890'
      name: ca-prod
      regions:
      - ca-central-1
      role: arn:aws:iam::172507017890:role/sre-ec2-role-assumed
    - account_id: '612646292843'
      name: contact-center
      regions:
      - us-east-1
      role: arn:aws:iam::612646292843:role/sre-ec2-role-assumed
```

A policy file will have a format like:

```
policies:
- name: a-name-you-create
  resource: actual resource name from cc documentation
  description: Some relevant description
  filters:
    - type: a type from cc resource documentation
  actions:
    - type: a type from cc resource documentation
```

Here is an example of a real policy file that removes some unused Security Group rules, adds one new rule and sends an email notification of all the Security Groups that were modified:

```
policies:
- name: mcafee-security-rules-rm-add
  resource: security-group
  description: Remove unused McAfee Security Group rules and add one new rule
  filters:
    - or:
      - type: ingress
        Ports: [445, 8085]
        Cidr:
          value: "10.16.10.174/32"
      - type: ingress
        Ports: [445, 8085]
        Cidr:
          value: "10.17.2.183/32"
      - type: ingress
        Ports: [445, 8085]
        Cidr:
          value: "10.17.2.182/32"
      - type: ingress
        Ports: [445, 8085]
        Cidr:
          value: "10.17.2.180/32"
      - type: ingress
        Ports: [443, 445, 8082, 8085]
        Cidr:
          value: "10.17.2.249/32"
  actions:
    - type: set-permissions
      # remove the permission matched by a previous ingress filter.
      remove-ingress: matched
      # add a list of permissions to the group.
      add-ingress:
        - IpPermissions:
          - IpProtocol: TCP
            FromPort: 443
            ToPort: 443
            IpRanges:
              - Description: allow McAfee EPO access from spw099epo02-new
                CidrIp: "10.17.2.249/32"
          - IpProtocol: TCP
            FromPort: 445
            ToPort: 445
            IpRanges:
              - Description: allow McAfee EPO access from spw099epo02-new
                CidrIp: "10.17.2.249/32"
          - IpProtocol: TCP
            FromPort: 8082
            ToPort: 8082
            IpRanges:
              - Description: allow McAfee EPO access from spw099epo02-new
                CidrIp: "10.17.2.249/32"
          - IpProtocol: TCP
            FromPort: 8085
            ToPort: 8085
            IpRanges:
              - Description: allow McAfee EPO access from spw099epo02-new
                CidrIp: "10.17.2.249/32"
    - type: notify
      template: default.html
      template_format: html
      priority_header: '1'
      subject: "{{ account }} McAfee Security Group Rule Modification (old removed and one new added)"
      to:
        - brian.gaber@compucom.com
      transport:
        type: sqs
        queue: https://sqs.us-east-1.amazonaws.com/472510080448/cloud-custodian
```