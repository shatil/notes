# Notes

# AWS

## EC2

### Resize Root Partition
You can resize (in this case, just grow) an EC2 Instance's root EBS volume
using EC2 Console, and
[then](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html):

Before:

```
[ec2-user@ip-10-10-10-10 ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs         16G   80K   16G   1% /dev
tmpfs            16G     0   16G   0% /dev/shm
/dev/nvme0n1p1   32G   25G  7.0G  78% /
```

Running:

```
[ec2-user@ip-10-10-10-10 ~]$ sudo growpart /dev/nvme0n1 1
CHANGED: disk=/dev/nvme0n1 partition=1: start=4096 old: size=67104734,end=67108830 new: size=2097147870,end=2097151966

[ec2-user@ip-10-10-10-10 ~]$ sudo resize2fs /dev/nvme0n1p1
resize2fs 1.43.5 (04-Aug-2017)
Filesystem at /dev/nvme0n1p1 is mounted on /; on-line resizing required
old_desc_blocks = 2, new_desc_blocks = 63
The filesystem on /dev/nvme0n1p1 is now 262143483 (4k) blocks long.
```

Use `xfs_growfs -d /` instead of `resize2fs` if you're on Amazon Linux 2.

After:

```
[ec2-user@ip-10-10-10-10 ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs         16G   80K   16G   1% /dev
tmpfs            16G     0   16G   0% /dev/shm
/dev/nvme0n1p1  985G   25G  960G   3% /
```

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

# Chef

## Attributes

### Update Node Attributes from EC2 Metadata
[EC2
Metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)
is useful for templates requiring details like `instanceId`.

```ruby
require 'json'
require 'net/http'

# Some metadata from EC2.
client = Net::HTTP.new('169.254.169.254', 80)
client.open_timeout = 0.5  # running outside AWS, this'll time out
client.read_timeout = 0.5  # running outside AWS, this'll time out
begin
  # Body looks like:
  # $ curl -w '\n' http://169.254.169.254/latest/dynamic/instance-identity/document
  #    {
  #      "accountId" : "123456789010",
  #      "architecture" : "x86_64",
  #      "availabilityZone" : "us-east-3c",
  #      "billingProducts" : null,
  #      "devpayProductCodes" : null,
  #      "imageId" : "ami-deadbeef"
  #      "instanceId" : "i-0cfe83617566afd9e",
  #      "instanceType" : "m4.large",
  #      "kernelId" : null,
  #      "pendingTime" : "2017-04-27T15:29:50Z",
  #      "privateIp" : "10.0.0.10",
  #      "ramdiskId" : null,
  #      "region" : "us-east-3",
  #      "version" : "2010-08-31",
  #    }
  body = client.get('/latest/dynamic/instance-identity/document').body
  document = JSON.parse(body)
rescue Errno::EHOSTUNREACH, JSON::ParserError, Net::OpenTimeout, Net::ReadTimeout
  document = {}
end
default.update(document)
default['region'] = document.fetch('region', 'us-east-3')
```

Public IP from EC2 Metadata:

```ruby
begin
  addr = client.get('/latest/meta-data/public-ipv4').body
rescue Net::OpenTimeout, Net::ReadTimeout, Errno::EHOSTUNREACH
  addr = 'localhost'
end
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

## Tick with initial burst
[Gist has
details](https://gist.github.com/shatil/0ca9fd3cf93827c65ba6378324f72c75) on
how to throttle or limit concurrency by time _with_ an initial burst.

```go
func TickBurst(interval time.Duration, size int) <-chan time.Time {
	c := make(chan time.Time, size)
	for i := 0; i < size; i++ {
		c <- time.Now()
	}
	go func() {
		t := time.Tick(interval)
		for {
			c <- <-t
		}
	}()
	return c
}
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

`brew install pipenv` and get an environment going for a project with the
following, which will create the necessary files for re-install of
version-locked modules:

```
pipenv install coloredlogs
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

```python
import ruamel.yaml
```

### Load and Preserve Comments in YAML file

```python
with open(fp) as f:
    d = ruamel.yaml.round_trip_load(f, preserve_quotes=True)
```

### Dump and Preserve Comments and Lines in YAML
This works great for things like `serverless.yml` files. For CloudFormation, I
think using the built-in YAML conversion is a better idea.

```python
with open(fp, "w") as f:
    yml = yaml.YAML()
    yml.indent(mapping=2, sequence=4, offset=2)
    yml.dump(d, stream=f)
```

## `argparse`
[Show defaults](https://stackoverflow.com/a/12151325/11536029) for `--help`
with `argparse`:

```python
argparse.ArgumentParser(
    formatter_class=argparse.ArgumentDefaultsHelpFormatter
)
```

### Sub-commands
`git` sub-command-like CLIs using subparsers is possible with Python.

```python
p = argparse.ArgumentParser()
s = p.add_subparsers(dest="choice", title="choice", required=True)
action = s.add_parser("action")
action.add_argument("--dry-run", action="store_true")
```

`p`, `s`, and `action` above [all support their own
description](https://docs.python.org/3/library/argparse.html#sub-commands),
etc.

Use conditionals to figure out what to do:

```python
args = p.parse_args()
if args.choice == "action":
    if args.dry_run:
        pass
```

## `logging`
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
