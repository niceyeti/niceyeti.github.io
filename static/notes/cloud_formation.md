# CloudFormation

CloudFormation templates provide declarative IaC.
Note: "CloudFormation" is both a service and the template definition; the term must be disambiguated.

SAM is an extension for CloudFormation that wraps it to hide a bunch of standardized complexity.

For resource definitions, see the [AWS Resource Types Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html).

* versioned controlled templates
* resource identifiers in one's stack enable cost-analysis
* automation: automate the creation/destruction of stacks dynamically for CD, or cost-minimization (e.g., destroy dev stacks every day at 5pm)

Creation:

* templates are uploaded to S3 and then referenced in CloudFormation (the service)
* managed by CloudFormation service
* be keen on creation policies, PermissionBoundaries, Roles, and security reqs

### Core Concepts

* Stacks: these are instances/realizations of resources created from a template
* StackSets: multiple stacks created across accounts or parametrically
* ChangeSets: the diff created when one creates a stack

## Components

* **Resources**: AWS resources

    ```yaml
    <service-provider::service-name::data-type-name>
    ...
    Type: AWS::EC2::Instance
    Type: AWS::ACMPCA::CertificateAuthority
    ...
    ```

* **Parameters**: the dynamic inputs of your template. These can be native types like number, string, list, or AWS-specific type related to their usecases. They can have range/property parameters definining their behavior.

Parameters are passed via file to the cli using the `--parameters-override` option; I assume that parameter inputs are exposed on the UI in the console.

    ```yaml
    Parameters:
      InstanceTypeParameter:
        Type: String
          Default: t2.micro
          AllowedValues:
            - t2.micro
            - m1.small
            - m1.large
          Description: Enter t2.micro, m1.small, or m1.large.
            Default is t2.micro.
    ```

* Pseudo-parameters: these are AWS specific parameters with fixed definitions:

    ```yaml
        AWS::Region
        AWS::AccountId
        AWS::StackId
        AWS::StackName
    ```

* **Metadata**: a section for arbitrary json or yaml definitions and labels, other information.

* **Mappings**: the static variables of your template. These have a map name, top level key, and second level key, and value: name -> top-level key -> second-level key -> value.

```yaml
Mappings: 
    RegionMap: 
        us-east-1: 
            "HVM64": "ami-0ff8a91507f77f867"
        us-west-1: 
            "HVM64": "ami-0bdb828fd58c52235"
...
Resources:
    myEC2Instance: 
        Type: "AWS::EC2::Instance"
        Properties: 
        ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", HVM64]
```

* **Outputs**: the outputs (references) of your stack for use in cross-stack references imported into other stacks. These can be returned in describe-stack calls, for example. You cannot delete a stack if its outputs are referenced in another stack. **Output names must be unique for a region**.

Outputs work great for separation of concerns: export a VpcId from one network stack for use in another, such that the user doesn't need to know about the details of the other stack.

Use `Fn::ImportValue` function to import another output.

```yaml
Outputs:
    BackupLoadBalancerDNSName:
    Description: The DNSName of the backup load balancer
        Value: !GetAtt BackupLoadBalancer.DNSName
        Condition: CreateProdResources  # Create output only if CreateProdResources is satisfied
    InstanceID:
    Description: The Instance ID
    Value: !Ref EC2Instance
    StackVPC:
    Description: The ID of the VPC
        Value: !Ref MyVPC
        Export:
        Name: !Sub "${AWS::StackName}-VPCID"
```

* **Conditionals**: composed of Rules and Assertions.
  * Rules: define when assertions take effect.
  * Assertions: the conditions/predicates (when, contains, equals, and, if...)

    Usecase example: "when environment=test, use modest resources instances (lightweight ec2, rds, etc)"

```yaml
    Rules:
      testInstanceType:
          RuleCondition: !Equals 
          - !Ref Environment
          - test
          Assertions:
          - Assert:
              'Fn::Contains':
                  - - a1.medium
                  - !Ref InstanceType
            AssertDescription: 'For a test environment, the instance type must be a1.medium'
      prodInstanceType:
          RuleCondition: !Equals 
          - !Ref Environment
          - prod
          Assertions:
          - Assert:
              'Fn::Contains':
                  - - a1.large
                  - !Ref InstanceType
            AssertDescription: 'For a production environment, the instance type must be a1.large'
```

* **Intrinsic Functions**: built-in functions for getting and resolving things.

    | Fn  | Example | Description |
    | --- | --- | --- |
    | Fn::Ref !Ref             | !Ref Foo | resolve a resource or parameter |
    | Fn::GetAttr !GetAttr     | "Fn::GetAtt" : [ "myELBLogicalId" , "DNSName" ]  | get an attribute of a resource |
    | Fn::FindInMap !FindInMap | "Fn::FindInMap" : [ "MapName", "TopLevelKey", "SecondLevelKey"] | find value in a map, via map name, top, and second-level keys |
    | Fn::ImportValue          | Fn::ImportValue: 'NetworkStackABC-SecurityGroupID' | resolve the value of an output of another stack |
    | Fn::Join                 | | join a string |
    | Fn::Sub !Sub             | 'Fn::Sub': '${AWS::StackName}-SubnetID' | substitute a value into a string, similar syntax to f-strings in python, `f"{some_value}:abc"` |

* **Creation policies and wait conditions**: policies by which you can set up creation/configuration relationships and resource dependencies.
  * CreationPolicy attribute: a hook to await creation based on anothe resource' creation.
  * DeleteionPolicy attribute: preserve or backup a resource when its stack is deleted.
  * DependsOn: configure creational dependencies between resources
  * WaitCondition: e.g., wait for the desired number of instances in a server cluster.
  * UpdatePolicy: configure a policy for when resources are updated.
