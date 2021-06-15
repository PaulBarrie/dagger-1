---
slug: /learn/108-cloudformation
---

# Dagger 108: provision infrastructure on AWS

In this guide you will learn how to automatically [provision infrastructure](https://dzone.com/articles/infrastructure-provisioning-–) on AWS, by integrating [Amazon Cloudformation](https://aws.amazon.com/cloudformation/) in your Dagger environment.

We will start with something simple: provisioning a new bucket on [Amazon S3](https://en.wikipedia.org/wiki/Amazon_S3). But Cloudformation can provision almost any AWS resource; and Dagger can integrate with the full Cloudformation API.

## Prerequisites

### Reminder

#### Guidelines

The provisioning strategy detailed below follows S3 best practices. In order to remain agnostic of your current AWS level, it deeply relies on S3 and Cloudformation documentation.

#### Relays

When developing a plan based on relays, the first thing to consider is to read their universe reference: it summarizes the expected inputs and their corresponding formats. [<u>Here</u>](https://dagger.io/aws/cloudformation) is the Cloudformation one.

### Setup

1.Initialize a new folder and a new workspace

```shell
mkdir infra-provisioning
cd ./infra-provisioning
dagger init
```

2.Create a new environment and create a `main.cue` file

```shell
dagger new s3-provisioning
cd ./.dagger/env/s3-provisioning/plan/ #Personal preference to directly work inside the plan
touch main.cue
```

3.Create corresponding `main` package

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main
```

## Cloudformation

Now that a plan has been set, let's implement the Cloudformation template and convert it to a Cue definition for further flexibility.

### Template creation

The idea here is to follow best practices in [<u>S3 buckets</u>](https://docs.aws.amazon.com/AmazonS3/latest/userguide/HostingWebsiteOnS3Setup.html) provisioning. Thanksfully, the AWS documentation contains a working [<u>Cloudformation template</u>](https://docs.aws.amazon.com/fr_fr/AWSCloudFormation/latest/UserGuide/quickref-s3.html#scenario-s3-bucket-website) that fits 95% of our needs.

1.Tweaking the template: removing some of the ouputs

The [<u>template</u>](https://docs.aws.amazon.com/fr_fr/AWSCloudFormation/latest/UserGuide/quickref-s3.html#scenario-s3-bucket-website) has far more outputs than necessary, as we just want to retrieve the bucket name:

import Tabs from "@theme/Tabs";
import TabItem from "@theme/TabItem";

<Tabs
  defaultValue="nv"
  values={[
    { label: 'Past Output', value: 'pv', },
    { label: 'New Output', value: 'nv', },
    { label: 'Full Base Template', value: 'ft', },
  ]
}>
<TabItem value="pv">

```json
"Outputs": {
    "WebsiteURL": {
        "Value": {
            "Fn::GetAtt": [
                "S3Bucket",
                "WebsiteURL"
            ]
        },
        "Description": "URL for website hosted on S3"
    },
    "S3BucketSecureURL": {
        "Value": {
            "Fn::Join": [
                "",
                [
                    "https://",
                    {
                        "Fn::GetAtt": [
                            "S3Bucket",
                            "DomainName"
                        ]
                    }
                ]
            ]
        },
        "Description": "Name of S3 bucket to hold website content"
    }
}
```

</TabItem>
<TabItem value="nv">

```json
"Outputs": {
    "Name": {
        "Value": {
            "Fn::GetAtt": [
                "S3Bucket",
                "Arn"
            ]
        },
        "Description": "Name S3 Bucket"
    }
}
```

</TabItem>
<TabItem value="ft">

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Resources": {
    "S3Bucket": {
      "Type": "AWS::S3::Bucket",
      "Properties": {
        "AccessControl": "PublicRead",
        "WebsiteConfiguration": {
          "IndexDocument": "index.html",
          "ErrorDocument": "error.html"
        }
      },
      "DeletionPolicy": "Retain"
    },
    "BucketPolicy": {
      "Type": "AWS::S3::BucketPolicy",
      "Properties": {
        "PolicyDocument": {
          "Id": "MyPolicy",
          "Version": "2012-10-17",
          "Statement": [
            {
              "Sid": "PublicReadForGetBucketObjects",
              "Effect": "Allow",
              "Principal": "*",
              "Action": "s3:GetObject",
              "Resource": {
                "Fn::Join": [
                  "",
                  [
                    "arn:aws:s3:::",
                    {
                      "Ref": "S3Bucket"
                    },
                    "/*"
                  ]
                ]
              }
            }
          ]
        },
        "Bucket": {
          "Ref": "S3Bucket"
        }
      }
    }
  },
  "Outputs": {
    "Name": {
      "Value": {
        "Fn::GetAtt": ["S3Bucket", "Arn"]
      },
      "Description": "Name S3 Bucket"
    }
  }
}
```

</TabItem>
</Tabs>

2.Some _"Pro tips"_

Double-checks at the template level can be done with manual uploads on Cloudformation's web interface or by executing the below command locally:

```shell
aws cloudformation validate-template  --template-body file://template.json
```

> PS: The _"Full Base Template"_ tab contains the base template used for the following parts of the guide

### JSON / YAML to Cue conversion

Once you'll get used to Cue, you might directly write Cloudformation templates in this language. As most of the current examples are either written in JSON or in YAML, let's see how to lazily convert them in Cue (optional but recommended).

#### 1. Modify main.cue

We will temporarly modify `main.cue` to process the conversion

<Tabs
  defaultValue="sv"
  values={[
    { label: 'JSON Generic Code', value: 'sv', },
    { label: 'YAML Generic Code', value: 'yv', },
    { label: 'JSON Full example', value: 'fv', },
  ]
}>
<TabItem value="sv">

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main
import "encoding/json"

point: json.Unmarshal(data)
data: #"""
// Paste above final JSON template here
"""#
```

</TabItem>
<TabItem value="fv">

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main

import (
  "encoding/json"
)

point: json.Unmarshal(data)
data: #"""
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Resources": {
        "S3Bucket": {
            "Type": "AWS::S3::Bucket",
            "Properties": {
                "AccessControl": "PublicRead",
                "WebsiteConfiguration": {
                    "IndexDocument": "index.html",
                    "ErrorDocument": "error.html"
                }
            },
            "DeletionPolicy": "Retain"
        },
        "BucketPolicy": {
            "Type": "AWS::S3::BucketPolicy",
            "Properties": {
                "PolicyDocument": {
                    "Id": "MyPolicy",
                    "Version": "2012-10-17",
                    "Statement": [
                        {
                            "Sid": "PublicReadForGetBucketObjects",
                            "Effect": "Allow",
                            "Principal": "*",
                            "Action": "s3:GetObject",
                            "Resource": {
                                "Fn::Join": [
                                    "",
                                    [
                                        "arn:aws:s3:::",
                                        {
                                            "Ref": "S3Bucket"
                                        },
                                        "/*"
                                    ]
                                ]
                            }
                        }
                    ]
                },
                "Bucket": {
                    "Ref": "S3Bucket"
                }
            }
        }
    },
    "Outputs": {
        "Name": {
            "Value": {
                "Fn::GetAtt": [
                    "S3Bucket",
                "Arn"
                ]
            },
            "Description": "Name S3 Bucket"
        }
    }
}
"""#
```

</TabItem>
<TabItem value="yv">

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main
import "encoding/yaml"

point: yaml.Unmarshal(data)
data: """
// Paste YAML here
"""
```

</TabItem>
</Tabs>

#### 2. Retrieve the Unmarshalled JSON

Then, still in the same folder, query the `point` value to retrieve the Unmarshalled result of `data`:

<Tabs
  defaultValue="sc"
  values={[
    { label: 'Short Command', value: 'sc', },
    { label: 'Full Base Output', value: 'fo', },
  ]
}>
<TabItem value="fo">

```shell
dagger query point
# Output:
# {
#   "AWSTemplateFormatVersion": "2010-09-09",
#   "Outputs": {
#     "Name": {
#       "Description": "Name S3 Bucket",
#       "Value": {
#         "Fn::GetAtt": [
#           "S3Bucket",
#           "Arn"
#         ]
#       }
#     }
#   },
#   "Resources": {
#     "BucketPolicy": {
#       "Properties": {
#         "Bucket": {
#           "Ref": "S3Bucket"
#         },
#         "PolicyDocument": {
#           "Id": "MyPolicy",
#           "Statement": [
#             {
#               "Action": "s3:GetObject",
#               "Effect": "Allow",
#               "Principal": "*",
#               "Resource": {
#                 "Fn::Join": [
#                   "",
#                   [
#                     "arn:aws:s3:::",
#                     {
#                       "Ref": "S3Bucket"
#                     },
#                     "/*"
#                   ]
#                 ]
#               },
#               "Sid": "PublicReadForGetBucketObjects"
#             }
#           ],
#           "Version": "2012-10-17"
#         }
#       },
#       "Type": "AWS::S3::BucketPolicy"
#     },
#     "S3Bucket": {
#       "DeletionPolicy": "Retain",
#       "Properties": {
#         "AccessControl": "PublicRead",
#         "WebsiteConfiguration": {
#           "ErrorDocument": "error.html",
#           "IndexDocument": "index.html"
#         }
#       },
#       "Type": "AWS::S3::Bucket"
#     }
#   }
# }
```

</TabItem>
<TabItem value="sc">

```shell
dagger query point
# Output:
# {
    #Prints value stored in point key
# }
```

</TabItem>
</Tabs>

#### 3. Store the output

This Cue version of the JSON template is going to be integrated inside our provisioning plan. Save the output for the next steps of the guide.

## Personal plan

With the Cloudformation template now finished, tested and converted in Cue. We can now enter the last part of our guide: piping everything together inside our personal plan.

Before continuing, don't forget to reset your `main.cue` plan to it's _Setup_ form:

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main
```

### Cloudformation relay

As our plan relies on [<u>Cloudformation's relay</u>](https://dagger.io/aws/cloudformation), let's dissect the expected inputs by gradually incorporating them in our plan.

| Name               |                   Type                    |                             Description                              |
| ------------------ | :---------------------------------------: | :------------------------------------------------------------------: |
| _config.region_    |                 `string`                  |                              AWS region                              |
| _config.accessKey_ |             `dagger.#Secret`              |                            AWS access key                            |
| _config.secretKey_ |             `dagger.#Secret`              |                            AWS secret key                            |
| _source_           |                 `string`                  |       Source is the Cloudformation template (JSON/YAML string)       |
| _stackName_        |                 `string`                  |                Stackname is the cloudformation stack                 |
| _onFailure_        | `*"DO_NOTHING" \| "ROLLBACK" \| "DELETE"` |           Behavior when failure to create/update the Stack           |
| _timeout_          |            `*10 \| \>=0 & int`            | Timeout for waiting for the stack to be created/updated (in minutes) |
| _neverUpdate_      |             `*false \| bool`              |               Never update the stack if already exists               |

1.General insights

As seen before in the documentation, values starting with `*` are default values. However, as a plan developer, we may face the need to add default values to inputs from relays that don't have one : Cue gives you this flexibility (cf. `config` value detailed below).

> WARNING: All inputs without a default option have to be filled for a proper execution of the relay. In our case:
>
> - _config.region_
> - _config.accessKey_
> - _config.secretKey_
> - _source_
> - _stackName_

2.The config value

The config values are all part of the `aws` relay. Regarding this package, as you can see above, none of the 3 required inputs contain default options.

For the sake of the exercise, let's say that our company's policy is to mainly deploy on the `us-east-2` region. Having this value set as a default option could be a smart and efficient decision for our dev teams. Let's see how to implement it:

<Tabs
  defaultValue="av"
  values={[
    { label: 'Before', value: 'bv', },
    { label: 'After', value: 'av', },
    { label: 'Details', value: 'dv', },
  ]
}>
<TabItem value="bv">

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main
```

</TabItem>
<TabItem value="av">

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main

import (
  "dagger.io/aws"
)

// AWS account: credentials and region
awsConfig: aws.#Config & {
  region: *"us-east-2" | string @dagger(input)
}
```

</TabItem>
<TabItem value="dv">

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main

import (
  "dagger.io/aws" // <-- Import AWS relay to instanciate aws.#Config
)

// AWS account: credentials and region
awsConfig: aws.#Config & { // Assign an aws.#Config definition to a field named `awsConfig`
    // awsConfig will be a directly requestable key : `dagger query awsConfig`
    // awsConfig sets the region to either an input, or a default string: "us-east-2"
  region: *"us-east-2" | string @dagger(input)
    // As we declare an aws.#Config, Dagger/Cue will automatically know that some others values inside this definition
    // are inputs, especially secrets (AccessKey, secretKey). Due to the confidential nature of secrets, we won't declare default values to them
}
```

</TabItem>
</Tabs>

Pro tips: In order to check wether it worked or not, these two commands might help

<Tabs
  defaultValue="fc"
  values={[
    { label: 'First command', value: 'fc', },
    { label: 'Second command', value: 'sc', },
    { label: 'Failed execution', value: 'fe', },
  ]
}>
<TabItem value="fc">

```shell
dagger input list # List required input in our personal plan
# Output:
# Input                Value                  Set by user  Description
# awsConfig.region     *"us-east-2" | string  false        AWS region
# awsConfig.accessKey  dagger.#Secret         false        AWS access key
# awsConfig.secretKey  dagger.#Secret         false        AWS secret key
```

</TabItem>
<TabItem value="sc">

```shell
dagger query # Query values / inspect default values (Very useful in case of conflict)
# Output:
# {
#   "awsConfig": {
#     "region": "us-east-2"
#   }
# }
```

</TabItem>
<TabItem value="fe">

```shell
dagger up # Try to run the plan. As expected, we encounter a failure
# Output:
# 9:07PM ERR system | required input is missing    input=awsConfig.accessKey
# 9:07PM ERR system | required input is missing    input=awsConfig.secretKey
# 9:07PM FTL system | some required inputs are not set, please re-run with `--force` if you think it's a mistake    missing=0s
```

</TabItem>
</Tabs>

Inside the `firstCommand` tab, we see that the `awsConfig.region` key has a default value set. It wasn't the case when we just imported the base relay.

Furthemore, in the `Failed execution` tab, the execution of the `dagger up` command fails because of the unspecified secret inputs.

3.Integrating Cloudformation relay

Now that we have the `config` definition properly configured, we can now import the Cloudformation one, and properly fill it :

<Tabs
  defaultValue="av"
  values={[
    { label: 'Before', value: 'bv', },
    { label: 'After', value: 'av', },
    { label: 'Details', value: 'dv', },
    { label: 'Full Base version', value: 'fv', },
  ]
}>
<TabItem value="bv">

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main

import (
  "dagger.io/aws"
)

// AWS account: credentials and region
awsConfig: aws.#Config & {
  region: *"us-east-2" | string @dagger(input)
}
```

</TabItem>
<TabItem value="av">

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main

import (
  "dagger.io/aws"
    "dagger.io/random"
    "dagger.io/aws/cloudformation"
)

// AWS account: credentials and region
awsConfig: aws.#Config & {
  region: *"us-east-2" | string @dagger(input)
}


// Create a random suffix
suffix: random.#String & {
  seed: ""
}

// Request the cloudformation stackname as an input, or generated a default one with a random suffix to keep uniqueness
cfnStackName: *"stack-\(suffix.out)"  | string @dagger(input) // Has to be unique

// AWS Cloudformation stdlib
cfnStack: cloudformation.#Stack & {
  config:    awsConfig
  stackName: cfnStackName
  onFailure: "DO_NOTHING"
  source:    json.Marshal(#cfnTemplate)
}

#cfnTemplate: {
    // Paste Cue Cloudformation template here
}
```

</TabItem>
<TabItem value="dv">

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main

import (
  "dagger.io/aws" // <-- Import AWS relay to instanciate aws.#Config
    "dagger.io/random" // <-- Import Random relay to instanciate random.#String
    "dagger.io/aws/cloudformation" // <-- Import Cloudformation relay to instanciate aws.#Cloudformation
)

// AWS account: credentials and region
awsConfig: aws.#Config & { // Assign an aws.#Config definition to a field named `awsConfig`
    // awsConfig will be a directly requestable key : `dagger query awsConfig`
    // awsConfig sets the region to either an input, or a default string: "us-east-2"
  region: *"us-east-2" | string @dagger(input)
    // As we declare an aws.#Config, Dagger/Cue will automatically know that some others values inside this definition
    // are inputs, especially secrets (AccessKey, secretKey). Due to the confidential nature of secrets, we won't declare default values to them
}

// AWS Cloudformation stdlib
cfnStack: cloudformation.#Stack & { // Assign an aws.#Cloudformation definition to a field named `cfnStack`
    // This definition is the stdlib package to use in order to deploy AWS instances programmatically

  config:    awsConfig // As seen in the relay doc, 3 config fields have to be provided : `config.region`, `config.accessKey` and `config.secretKey`
    // As their names contain a `.`, it means that the value `config` expects 3 fields `region`, `accessKey` and `secretKey`, included in a `aws.#Config` parent definition

    stackName: cfnStackName // We assign to the `stackName` the `cfnStackName` declared below.
    // `stackName` expects a string type. However, as a plan developer, we wanted to give the developer a choice : either a default random value, or an input
    // The default random value *"stack-\(suffix.out)" uses the random.#String relay to generate a random value. We append it's result inside `"\(append_happening_here)"`

  onFailure: "DO_NOTHING" // As cited in the Cloudformation relay, the `onFailure` key defines Cloudformation's stack behavior on failure

  source:    json.Marshal(#cfnTemplate) // source expects a JSON artifact. Here we remarshall the template decaled in Cue
}

// Create a random suffix (cf. random relay)
suffix: random.#String & { // Assign a #random definition to a field named `suffix`
  seed: "" // Set seed to empty string, to generate a new random string every time
} // Output -> suffix.out is a random string

// Request the cloudformation stackname as an input, or generated a default one with a random suffix to keep uniqueness
cfnStackName: *"stack-\(suffix.out)"  | string @dagger(input) // Has to be unique

#cfnTemplate: {
    // Paste Cue Cloudformation template here
}
```

</TabItem>
<TabItem value="fv">

```cue title=".dagger/env/s3-provisioning/plan/main.cue"
package main

import (
  "encoding/json"

  "dagger.io/aws"
    "dagger.io/random"
  "dagger.io/aws/cloudformation"
)

// AWS account: credentials and region
awsConfig: aws.#Config & {
  region: *"us-east-2" | string @dagger(input)
}

// Create a random suffix
suffix: random.#String & {
  seed: ""
}

// Query the Cloudformation stackname, or create one with a random suffix to keep unicity
cfnStackName: *"stack-\(suffix.out)" | string @dagger(input)

// AWS Cloudformation stdlib
cfnStack: cloudformation.#Stack & {
  config:    awsConfig
  stackName: cfnStackName
  onFailure: "DO_NOTHING"
  source:    json.Marshal(#cfnTemplate)
}

#cfnTemplate: {
  "AWSTemplateFormatVersion": "2010-09-09",
  "Outputs": {
    "Name": {
      "Description": "Name S3 Bucket",
      "Value": {
        "Fn::GetAtt": [
          "S3Bucket",
          "Arn"
        ]
      }
    }
  },
  "Resources": {
    "BucketPolicy": {
      "Properties": {
        "Bucket": {
          "Ref": "S3Bucket"
        },
        "PolicyDocument": {
          "Id": "MyPolicy",
          "Statement": [
            {
              "Action": "s3:GetObject",
              "Effect": "Allow",
              "Principal": "*",
              "Resource": {
                "Fn::Join": [
                  "",
                  [
                    "arn:aws:s3:::",
                    {
                      "Ref": "S3Bucket"
                    },
                    "/*"
                  ]
                ]
              },
              "Sid": "PublicReadForGetBucketObjects"
            }
          ],
          "Version": "2012-10-17"
        }
      },
      "Type": "AWS::S3::BucketPolicy"
    },
    "S3Bucket": {
      "DeletionPolicy": "Retain",
      "Properties": {
        "AccessControl": "PublicRead",
        "WebsiteConfiguration": {
          "ErrorDocument": "error.html",
          "IndexDocument": "index.html"
        }
      },
      "Type": "AWS::S3::Bucket"
    }
  }
}
```

</TabItem>
</Tabs>

### Deploy

Finally ! We now have a working template ready to be used to provision S3 infrastructures. Let's add the missing inputs (aws credentials) and let's deploy it :

<Tabs
  defaultValue="nd"
  values={[
    { label: 'Normal deploy', value: 'nd', },
    { label: 'Debug deploy', value: 'dd', },
  ]
}>
<TabItem value="nd">

```shell
dagger input secret awsConfig.accessKey yourAccessKey

dagger input secret awsConfig.secretKey yourSecretKey

dagger input list
# Input                 Value                  Set by user  Description
# awsConfig.region      *"us-east-2" | string  false        AWS region
# awsConfig.accessKey   dagger.#Secret         true         AWS access key <-- Specified
# awsConfig.secretKey   dagger.#Secret         true         AWS secret key <-- Specified
# suffix.length         *12 | number           false        length of the string
# cfnStack.timeout      *10 | >=0 & int        false        Timeout for waiting for the stack to be created/updated (in minutes)
# cfnStack.neverUpdate  *false | bool          false        Never update the stack if already exists

# All the other inputs have default values, we're good to go !

dagger up
# Output:
#2:22PM INF suffix.out | computing
#2:22PM INF suffix.out | completed    duration=200ms
#2:22PM INF cfnStack.outputs | computing
#2:22PM INF cfnStack.outputs | #15 1.304 {
#2:22PM INF cfnStack.outputs | #15 1.304     "Parameters": []
#2:22PM INF cfnStack.outputs | #15 1.304 }
#2:22PM INF cfnStack.outputs | #15 2.948 {
#2:22PM INF cfnStack.outputs | #15 2.948     "StackId": "arn:aws:cloudformation:us-east-2:817126022176:stack/stack-emktqcfwksng/207d29a0-cd0b-11eb-aafd-0a6bae5481b4"
#2:22PM INF cfnStack.outputs | #15 2.948 }
#2:22PM INF cfnStack.outputs | completed    duration=35s

dagger output list
# Output                 Value                                                    Description
# suffix.out             "emktqcfwksng"                                           generated random string
# cfnStack.outputs.Name  "arn:aws:s3:::stack-emktqcfwksng-s3bucket-9eiowjs1jab4"  -
```

</TabItem>
<TabItem value="dd">

```shell
dagger input secret awsConfig.accessKey yourAccessKey

dagger input secret awsConfig.secretKey yourSecretKey

dagger input list
# Input                 Value                  Set by user  Description
# awsConfig.region      *"us-east-2" | string  false        AWS region
# awsConfig.accessKey   dagger.#Secret         true         AWS access key <-- Specified
# awsConfig.secretKey   dagger.#Secret         true         AWS secret key <-- Specified
# suffix.length         *12 | number           false        length of the string
# cfnStack.timeout      *10 | >=0 & int        false        Timeout for waiting for the stack to be created/updated (in minutes)
# cfnStack.neverUpdate  *false | bool          false        Never update the stack if already exists

# All the other inputs have default values, we're good to go !

dagger up -l debug
#Output:
# 3:50PM DBG system | detected buildkit version    version=v0.8.3
# 3:50PM DBG system | spawning buildkit job    localdirs={
#     "/tmp/infra-provisioning/.dagger/env/infra/plan": "/tmp/infra-provisioning/.dagger/env/infra/plan"
# } attrs=null
# 3:50PM DBG system | loading configuration
# ... Lots of logs ... :-D
# Output                 Value                                                    Description
# suffix.out             "abnyiemsoqbm"                                           generated random string
# cfnStack.outputs.Name  "arn:aws:s3:::stack-abnyiemsoqbm-s3bucket-9eiowjs1jab4"  -

dagger output list
# Output                 Value                                                    Description
# suffix.out             "abnyiemsoqbm"                                           generated random string
# cfnStack.outputs.Name  "arn:aws:s3:::stack-abnyiemsoqbm-s3bucket-9eiowjs1jab4"  -
```

</TabItem>
</Tabs>

> The deployment went well !

In case of a failure, the `Debug deploy` tab shows the command to use in order to get more informations.
The name of the provisioned S3 instance lies in the `cfnStack.outputs.Name` output key, without `arn:aws:s3:::`

> With this provisioning infrastructure, your dev team will easily be able to instanciate aws infrastructures : all they need to know is `dagger input list` and `dagger up`, isn't that awesome ? :-D

PS: This plan could be further extended with the AWS S3 example : it could not only provision an infrastructure but also easily deploy on it.

PS1: As it could make a nice first exercise for you, this won't be detailed here. However, we're interested in your imagination : let us know your implementations :-)