# Notes

# AWS

## IAM

### Policy Document for ECS Secrets
Required for decrypting secret environment variables by `ecsTaskExecutionRole`.
Using KMS's default key, so don't need to spcify "kms:Decrypt".

* https://docs.aws.amazon.com/AmazonECS/latest/userguide/task_execution_IAM_role.html#task-execution-secrets
* https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data.html#secrets-iam

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Secrets",
            "Effect": "Allow",
            "Action": ["ssm:GetParameter"],
            "Resource": ["arn:aws:ssm:*:*:parameter/ecs/*"]
        }
    ]
}
```

# Golang

## Iterate line-by-line over file

```go
scanner := bufio.NewScanner(f)
for scanner.Scan() {
	line := scanner.Text()  // might want to strings.TrimSpace()
}
if err := scanner.Err(); err != nil {
	log.Fatal(err)
}
```

Parsing long (like HTTP log) lines requires a larger buffer:

```go
// Start w/ min buf size 64 KiB and max to 512 MiB _per line_ to reduce chance of:
// "reading standard input: bufio.Scanner: token too long"
scanner.Buffer(make([]byte, 64*1024), 512*1024*1024)
```

## Trigger `url.Parse` error
I [learned how to cause `url.Parse` to produce an
error](https://golang.org/src/net/url/url_test.go) during testing:

```go
url.Parse("%")
```

# Python

## Dependency Management
I used to use `pip` or `pip3` to install dependencies on machines, or use
`virtualenv` to manage more complex, or conflicting, deps. I think moving
forward, [I'll be using Pipenv](https://pipenv.readthedocs.io/en/latest/).

## Logging
I like this Python logging configuration:

```python
logging.basicConfig(
    level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s"
)
```

It looks like:

```
2019-04-04 22:48:24,716 - INFO - Found credentials in shared credentials file: ~/.aws/credentials
```

## Type Hints
[Type hints cheat sheet (Python
3)](https://mypy.readthedocs.io/en/latest/cheat_sheet_py3.html)

## YAML
[`ruamel.yaml`](https://yaml.readthedocs.io/en/latest/pyyaml.html) support
round-trip loading and dumping while maintaning comments, and is more
feature-rich than `PyYAML`.

```
pip3 install ruamel.yaml==0.15.89  # YAML module that preserves comments
```

```
import ruamel.yaml
```

### Load and Preserve Comments in YAML file
```
with open(fp) as f:
    d = ruamel.yaml.round_trip_load(f, preserve_quotes=True)
```

### Dump and Preserve Comments and Lines in YAML
This works great for things like `serverless.yml` files. For CloudFormation, I
think using the built-in YAML conversion is a better idea.

```
with open(fp, "w") as f:
    yml = yaml.YAML()
    yml.indent(mapping=2, sequence=4, offset=2)
    yml.dump(d, stream=f)
```
