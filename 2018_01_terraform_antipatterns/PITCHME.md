## Terraform

### Best Practices and Anti Patterns

---
## What is Terraform?

 - Infrastructure Management NOT Configuration Management
 - Vendor-agnostic (comparing to AWS CloudFormation, Heat templates)
 - Works with other APIs besides cloud provider (mongo, elastic etc.)

---?image=assets/image/2018_01_terraform_antipatterns/churchill.jpg&opacity=100

---

## Terraform Anti-patterns

---

### Hardcode AWS credential variables in files

```
provider "aws" {
  access_key = "${var.aws_access_key}"
  secret_key = "${var.aws_secret_key}"
  region = "${var.aws_region}"
}

provider "aws" {
  region = "${var.aws_region}"
}
```

@[1-5](Hardcoding key and password limits auth types.)
@[7-9](Different providers support multiple ways of login. For aws - shared config, env, IAM roles.)

---

### Terraform as deployment or CM tool

- 
Use suitable tools like Spinnaker etc.
https://vinayaksb.files.wordpress.com/2016/01/deployment_pipeline-4.png

+++

Don't manage application part in TF

OR

Make terraform adjustable to remote changes

+++

```
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t2.micro"
}

data "aws_ami" "web" {
  most_recent = true

  filter {
    name   = "name"
    values = ["web_ami-*"]
  }

  filter {
    name   = "tag:Status"
    values = ["latest"]
  }

  owners = ["123456789"]
}

resource "aws_instance" "web" {
  ami           = "${data.aws_ami.web.id}"
  instance_type = "t2.micro"
}
```
@[1-4](Instead of hardcoding ami)
@[5-25](Make it adjastable. Deployment tool should mark latest artifact for terraform to find)

Note:
Антипаттерны терраформа - 1. использование терраформа как деплоймент тулы для приложений. Испольщовать для деплоймента инфраструктурных частей – например задеплоить ваулт, через кубернетс провайдер задеплоить демонсет датадога и тд - ОК. Деплоить кор приложения компании который происходят ежедневно – плохо. Для этого лучше использовать подходящие тулы – спиннакер и тд.

---

### Separate security/access/roles for every service

- Seems like a good idea at first |
- Hard to manage afterwards - imagine changing ssh access for every service |
- Access to managing security, firewalls, etc. should not be granted to everyone |

+++

- Identify common patterns, separate by environments
  + ssh from bastion hosts
  + https internal
  + https external
  + etc ...
- Create them in separate project and use `terraform_remote_state`

Note:
Антипаттерны тераформа – 2. а давайте у каждого сервиса все будет свое – иам группа, секьюрити группа и тд ведь это легко. Это приводит к хаосу, и усложняеет глобавльные изменения. Лучше выделить общие требования – доступ с бастион хоста, внутренний хттп/с, внешний хттп/с, какой-то общий доступ к с3 бакету и тд. Создавать  отдельные группы только для выделяющихся кейсов.

---

### Multiple environments in one place

- You can break prod while updating stage |
- Promoting changes is hard - copy pasting can lead to errors |

+++

#### HashiCorp way

- Use `workspace`s

Cons:
- Separate permissions are only available in Terraform Enterprise
- In Terraform OSS It creates weird directory structure with hardcoded `default` environment that you cant change

+++

Use partial remote state configuration to get different states for environments

```
terraform {
  backend "s3" {
    encrypt    = "true"
  }
}

terraform init -backend-config="key=terraform/${PWD##*/}/${AWS_REGION}.tfstate" \
  -backend-config="bucket=${ENVIRONMENT}"
```

+++
 - Use separate `TF_DATA_DIR` (available from 0.11, before 0.11 you can delete `.terrafrom` via wrapper)

```bash
# terraform > 0.11
export TF_DATA_DIR=".tf-${AWS_REGION}-${ENVIRONMENT}"
terraform init
terraform apply -var-file=stage.tfvars

export ENVIRONMENT="prod"

# terraform < 0.11
cleanup () {
  if [ -d ".terraform" ]; then
    rm -rf .terraform
  fi
}

run_terraform () {
   cleanup
   $TF_BIN init
   $TF_BIN $TERRAFORM_COMMAND
   cleanup
}
```

Note:
Антипаттерны терраформа  - 3. Использование terraform workspace  

---

### ~~Best~~ Good Practices ( subjective :)

Note:

---

### BYOT

#### Bring Your Own Terraform

- Package terraform and required tools in docker image
- Someone has already done it for you: [coveo/tgf](https://github.com/coveo/tgf)
- Create your own with [terraform-bundle](https://github.com/hashicorp/terraform/tree/master/tools/terraform-bundle)

Note:
Гуд практис 1 – носить терраформ с собой - упакованый докер имедж со своими провайдерами, нудными либами для экстернал провайдера и тд.

---

## Hidden gems

### Use external data source

```
data "external" "example" {
  program = ["python", "${path.module}/example-data-source.py"]

  query = {
    id = "abc123"
  }
}
```

+++

- Use for tricky data structures that are not possible to create in terraform
  + Multi layered maps
- For API calls its better to create your own provider. Good reason to learn Go!

Note:
Терраформ хак 3 – юзайте external data source provider. Но для того что бы генерить невозмодные для терраформа структуры данных. Для обращения к сервисам/апи лучше написать свой провайдер если нету.

---

## http data source

- Get data from arbitrary APIs

```
data "http" "example" {
  url = "https://checkpoint-api.hashicorp.com/v1/check/terraform"

  # Optional request headers
  request_headers {
    "Accept" = "application/json"
  }
}
```

---

## Useful links

- [Terraform Recommended Practices](https://www.terraform.io/docs/enterprise/guides/recommended-practices/index.html)

---
### Questions?

<br>

@fa[twitter gp-contact](@gordonbondon)

@fa[github gp-contact](gordonbondon)
