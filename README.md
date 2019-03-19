# Notes

# Python

## Dependency Management
I used to use `pip` or `pip3` to install dependencies on machines, or use
`virtualenv` to manage more complex, or conflicting, deps. I think moving
forward, [I'll be using Pipenv](https://pipenv.readthedocs.io/en/latest/).

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
