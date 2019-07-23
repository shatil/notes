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
