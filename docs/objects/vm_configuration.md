# VM configuration

The `VM configuration` is a convenient object which describes the combinations of all platforms and targets pairs that we need as virtual machines and each of those are going to be generated in the form of `VM definition` object by using the `parse_vms_definitions` role.

## Scheme 
This object has the following scheme:
```
permutations: 
  # targets names
  targets: [] 
  # platforms names
  platforms: []
list:
  - platform: platform name
    target: target name
  - platform: platform name
    target: target name
    template: template name
  - ...
  ...
```
### Properties

- `permutations` : lists all domains's value of `platforms` and `targets` that need to be permutated as list of (platform, target) pairs
- `list` : It's a list of all explicit declaration of some `(platform, target)` object pairs. It may contains also tuples in the `(platform, target, template)` form to assign a template name inside the `VM definition` if it requires to be used while installed.

## Usage
This object is used as variable parameter for the `parse_vms_definitions` role which will combine both the permutations and definitions lists and so will lookup, parse and merge both platform and target definitions for each pair of the combined list.

**Note**: foreach `(platform, target, template)` tuple the `parse_vms_definitions` role will set the template name to the `vm.metadata.template` property.