# Notes

## AWS

### EC2

#### Resize Root Partition

You can resize (in this case, just grow) an EC2 Instance's root EBS volume
using EC2 Console, and
[then](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html):

Before:

```bash
[ec2-user@ip-10-10-10-10 ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs         16G   80K   16G   1% /dev
tmpfs            16G     0   16G   0% /dev/shm
/dev/nvme0n1p1   32G   25G  7.0G  78% /
```

Running:

```bash
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

```bash
[ec2-user@ip-10-10-10-10 ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs         16G   80K   16G   1% /dev
tmpfs            16G     0   16G   0% /dev/shm
/dev/nvme0n1p1  985G   25G  960G   3% /
```

### IAM

#### Policy Document for ECS Secrets

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

## `troposphere`

Produce nice-looking YAML:

```python
template.to_yaml(clean_up=True)
```

## Chef

### Attributes

#### Update Node Attributes from EC2 Metadata

[EC2
Metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)
is useful for templates requiring details like `instanceId`.

```ruby
require 'json'
require 'net/http'

## Some metadata from EC2.
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

## Git

Modifying Git history is possible, and Git supplies some really useful filters
to accomplish this. Changes apply starting from and "earlier" commit and going
towards a more recent commit, i.e., `HEAD`. Impact? If I rename a folder, I'll
have to check for its existence or the Git history editing will fail.

Remember to operate on a fresh clone and not lose work.

### Delete files from Git history

```bash
$ git filter-branch --tree-filter 'rm -rf references salary  shaatibiyyah' HEAD
Rewrite 54741c76bb97e699f0daa63a6ee32e955c0a97e6 (34/63) (1 seconds passed, remaining 0 predicted)
Ref 'refs/heads/master' was rewritten
```

This expunges the specified files or directories from Git history. This
`--tree-filter` command can be something else, e.g., `mv`. Just the [first
example](https://dalibornasevic.com/posts/2-permanently-remove-files-and-folders-from-a-git-repository)
worked.

If you run `filter-branch` again, you'll get an error because there's a backup!
So run it with `-f`.

```text
Cannot create a new backup.
A previous backup already exists in refs/original/
Force overwriting the backup with -f
```

### Move or rename files while preserving Git history

If using a command like `mv` that fails if something is missing, check for its
existence. [Example from StackOverflow](https://stackoverflow.com/a/15135004):

```bash
$ git filter-branch -f --tree-filter \
    'if [ -d letters ]; then mv letters/* .; rmdir letters; fi' \
    HEAD
Rewrite 55b23698eae0b1761c7122c43e428e588922e1d9 (38/63) (1 seconds passed, remaining 0 predicted)
Ref 'refs/heads/master' was rewritten
```

### Prune empty commits from Git history

Delete or remove commits that are empty (maybe as a result of repo surgery above?):

```bash
$ git filter-branch --prune-empty --tag-name-filter cat -- --all
Rewrite 71057ef757c335d07876099ede72d4ae82f3fbc3 (55/63) (1 seconds passed, remaining 0 predicted)
Ref 'refs/heads/master' was rewritten
Ref 'refs/remotes/origin/master' was rewritten
WARNING: Ref 'refs/remotes/origin/master' is unchanged
```

### Push existing code to new GitHub repo

Really this works for any Git hosting service.

```bash
git remote add origin git@github.com:shatil/letters.git
git push -u origin master
```

If there's no code but it's a new repo nonetheless:

```bash
echo "# letters" >> README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin git@github.com:shatil/letters.git
git push -u origin master
```

## GitHub

Visit account [Settings/Emails](https://github.com/settings/emails) to find
GitHub no-reply email address, then [set it for a single
repository](https://docs.github.com/en/github/setting-up-and-managing-your-github-user-account/setting-your-commit-email-address#setting-your-email-address-for-a-single-repository).

```bash
git config user.email "email@example.com"
```

Confirm:

```bash
git config user.email
```

### Repo surgery

Beware that modifying commits (e.g., fixing messages), while it looks on the
local repo like original dates are used, GitHub will not buy it.

Original:

```diff
commit a9f6cd75f4907856650b738f10070ef1010cba3d
Author: Shatil Rafiullah <16421632+shatil@users.noreply.github.com>
Date:   Wed Feb 8 18:17:39 2012 -0800

    Added some cosmetic edits to differentiate 'ta' from 'tA' and such for shaatibiyyah-22-fathi-wal-imaalah.
```

Reword using `git rebase -i` and selecting `reword`:

```diff
commit 9a852c0d38b824a12d7b3383978b8ee06165ab85
Author: Shatil Rafiullah <16421632+shatil@users.noreply.github.com>
Date:   Wed Feb 8 18:17:39 2012 -0800

    Added some cosmetic edits to differentiate 'ta' from 'tA'

    And related things for `shaatibiyyah-22-fathi-wal-imaalah`.
```

Result is pretty Git history locally, but sadly, it's not reflected in GitHub's
dates or comnit history.

## Golang

### Benchmark

Compare performance of implementation choices using Go's built-in support for
parameterized benchmarks:

```go
func BenchmarkSomething(b *testing.B) {
	benches := []struct {
		name     string
		function func(string) bool
		param    string
	}{
		{"og", oldFunc, "param1"},
		{"fresh", freshFunc, "freshParam"},
	}
	for _, bb := range benches {
		b.Run(bb.name, func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				bb.function(bb.param)
			}
		})
	}
}
```

Run _all_ benchmarks, or modify to target a few:

```zsh
go test -benchmem -run=^$ ./... -bench ^(BenchmarkSomething)$
```

### Data race

Pushing or popping via a `container/heap` like this causes a data race:

```go
heap.Push(&(shelf.overflow), order)
```

Which looks like this:

```bash
$ go test -cover -race -coverprofile=coverage.txt -covermode=atomic
==================
WARNING: DATA RACE
Read at 0x00c00019e0c0 by goroutine 16:
  github.com/shatil/kitchen-nightmare.Receive()
      ~/go/src/github.com/shatil/kitchen-nightmare/business-logic.go:23 +0x472

Previous write at 0x00c00019e0c0 by goroutine 19:
  github.com/shatil/kitchen-nightmare.(*Orders).Pop()
      ~/go/src/github.com/shatil/kitchen-nightmare/order.go:126 +0x10e
  github.com/shatil/kitchen-nightmare.(*Shelf).Pop()
      <autogenerated>:1 +0x43
  container/heap.Pop()
      /usr/local/Cellar/go/1.13/libexec/src/container/heap/heap.go:64 +0xb0
  github.com/shatil/kitchen-nightmare.(*Shelf).Get()
      ~/go/src/github.com/shatil/kitchen-nightmare/shelf.go:56 +0x1f7
  github.com/shatil/kitchen-nightmare.PickUp.func2()
      ~/go/src/github.com/shatil/kitchen-nightmare/business-logic.go:49 +0xdb
```

When `overflow` is something like:

```go
type Shelf struct {
	overflow []*Item
}
```

Fix:

```go
overflow := shelf.overflow
heap.Push(&overflow, order)
```

### Get file descriptor number for open file

It's of type `uintptr`, so may need to be cast to `int()`, like for
[`os.Stdin`](https://golang.org/pkg/os/#pkg-variables).

```go
int(os.Stdin.Fd())
```

### List of values for `flag`

Use [`flag.Var`](https://golang.org/pkg/flag/#Var) to create a CLI argument
that appends to a slice. Can `append` the original type!

```go
// Nargs represents some number of command line arguments.
type Nargs []string

// Set value of named CLI flag.
func (nargs *Nargs) Set(name, value string) error {
	*nargs = append(*nargs, value)
	return nil
}

// String comma-delimits members of Nargs.
func (nargs Nargs) String() string {
	return strings.Join([]string(nargs), ", ")
}
```

Then wherever `flag` is configured:

```go
n := Nargs{}
flag.Var(&n, "nargs", "append to list of arguments with each invocation")
```

### Iterate line-by-line over file

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

### Prefix lines written to `io.Writer` like `os.Stdout`

Go abstracts things writing to files with interfaces like `io.Writer` or
`io.ReadWriter`. Sometimes, lines written require a prefix. See it in
[action](https://play.golang.org/p/orR5ifLdPJZ) on Go Playground.


```go
// PrefixWriter writes a prefix before each line supplied.
type PrefixWriter struct {
	io.Writer
	Prefix      []byte
	Destination io.Writer
}

// Write to Destination io.Writer with a prefix before each line.
//
// '\n' line-endings are used, meaning '\r' is preserved for DOS line ends.
func (pw *PrefixWriter) Write(paragraph []byte) (written int, err error) {
	last := len(paragraph) - 1 // reduces ns/op
	line := pw.Prefix          // recycling reduces B/op and allocs/op
	position := 0

	for i, character := range paragraph {
		if character == '\n' || i == last {
			line = append(line[:len(pw.Prefix)], paragraph[position:i+1]...)
			n, err := pw.Destination.Write(line)
			if err != nil {
				return written, err
			}
			written += n - len(pw.Prefix)
			position = i + 1
		}
	}
	return
}
```

### Tick with initial burst
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

### Trigger `url.Parse` error
I [learned how to cause `url.Parse` to produce an
error](https://golang.org/src/net/url/url_test.go) during testing:

```go
url.Parse("%")
```

## Python

### Dependency Management

I used to use `pip` or `pip3` to install dependencies on machines, or use
`virtualenv` to manage more complex, or conflicting, deps. I think moving
forward, [I'll be using Pipenv](https://pipenv.readthedocs.io/en/latest/).

`brew install pipenv` and get an environment going for a project with the
following, which will create the necessary files for re-install of
version-locked modules:

```
pipenv install coloredlogs
```

### Type Hints

[Type hints cheat sheet (Python
3)](https://mypy.readthedocs.io/en/latest/cheat_sheet_py3.html)

### YAML

[`ruamel.yaml`](https://yaml.readthedocs.io/en/latest/pyyaml.html) support
round-trip loading and dumping while maintaning comments, and is more
feature-rich than `PyYAML`.

```
pip3 install ruamel.yaml==0.15.89  # YAML module that preserves comments
```

```python
import ruamel.yaml
```

#### Load and Preserve Comments in YAML file

```python
with open(fp) as f:
    d = ruamel.yaml.round_trip_load(f, preserve_quotes=True)
```

#### Dump and Preserve Comments and Lines in YAML

This works great for things like `serverless.yml` files. For CloudFormation, I
think using the built-in YAML conversion is a better idea.

```python
with open(fp, "w") as f:
    yml = yaml.YAML()
    yml.indent(mapping=2, sequence=4, offset=2)
    yml.dump(d, stream=f)
```

### `argparse`

[Show defaults](https://stackoverflow.com/a/12151325/11536029) for `--help`
with `argparse`:

```python
argparse.ArgumentParser(
    formatter_class=argparse.ArgumentDefaultsHelpFormatter
)
```

#### Sub-commands

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

### `logging`

I like this Python logging configuration:

```python
logging.basicConfig(
    level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s"
)
```

It looks like:

```bash
2019-04-04 22:48:24,716 - INFO - Found credentials in shared credentials file: ~/.aws/credentials
```
