
# Orion NPM Nornir Inventory

A Nornir inventory built from Orion NPM using ***orionsdk***. The inventory and accompanying script will do the following:

- Gathers NPM devices filtering the devices based on SQL query logic (*FROM* and *WHERE*)
- NPM device attributes (*SELECT*) are stored in the Nornir data dictionary as host_vars
- Builds device type groups based on *MachineType* which defines the Nornir *platform* (*connection_options*)
- Customisable SQL query (*WHERE*) and device attribute collection (*SELECT*)
- Filter the Nornir inventory at runtime using flags

## SolarWinds Query Language (SWQL)

[SWQL](https://user-images.githubusercontent.com/33333983/119037187-8eb88100-b9a9-11eb-9417-106d21eb7591.gif) is a proprietary, read-only subset of SQL which is used by the inventory plugin to query the SolarWinds database for specific network information. It uses standard SQL query logic of *SELECT … FROM … WHERE …* with SELECT and WHERE set in the default values (*npm_select* and *npm_where*) and FROM being an arbitrary value.

- **FROM:** Set by default to *'Orion.Nodes'* so that SELECT *'Caption'*, *'IPAddress'* and *'MachineType'* are always usable
- **SELECT:** An optional list of the device attributes to pull from Orion which are added as host data dictionaries. *Caption*, *IPAddress* and *MachineType* are explicit options set in the background with *Caption* used as the Nornir inventory *host*, *IPAddress* the *hostname* and *MachineType* defining group membership. Can use any pre-defined [schema](http://solarwinds.github.io/OrionSDK/schema/index.html) property or custom properties (must start with *Nodes.CustomProperties*).
- **WHERE:** A mandatory string used to filter the Nodes returned by the query. Although a string can be multiple conditional elements such as
*"Vendor = 'Cisco' and Nodes.Status = 1"* to match Cisco objects that are up.

## Inventory settings

The inventory settings hold the Orion and network device credentials as well as ASQL parameters used to filter to the orion database. The file structure is split into three parent dictionries with the settgins for each element underneath it.

| Parent | Variable | Type | Description |
| npm | -------- | -----| ----------- |
| npm | server | `string` | IP or hostname of the Orion NPM server will gather devices from
| npm | user | `string` | Username for orion, can be overriden at runtime with `-nu'
| npm | pword | 'string' | Optional password for orion
| npm | ssl_verify | `boolean` | Disables CA certificate validation warnings
| npm | select | `list` | Device (SWQL) attributes gathered and added to the nornir inventory (can be empty list)
| npm | where | `string` | Filter (SWQL) to define which Orion nodes to gather attributes from (such as vendor and/or status)
| groups | n/a | `list` | List of groups and filters based on *'MachineType'* to decide the group membership (can be empty list)
| device | user | `string` | Nornir inventory device username used when connecting to them, is same across all
| device | pword | `string` | Optional password for all devices

The passwords are only there for testing, at runtime if not set it will prompt for passwords, this is the preferable method.

The default inventory settings files (*inv_settings.yml*) has the following SWQL values and resulting logic.

**Filter (WHERE):** Gather device attributes for Cisco or Checkpoint devices (*Vendor*) that are up (*1)*

```yaml
npm:
  where: (Vendor = 'Cisco' or Vendor ='Check Point Software Technologies Ltd') and Nodes.Status = 1
```

**Attributes (SELECT):** Device attributes that will be gathered for each device. This will gather the default of *'Caption'*, *'IPAddress'* and *'MachineType'* (explict, dont need specifying) as well as *IOSversion* (Checkpoint has none) and custom attributes *Infra_Location* and *Infra_Logical_Location*.

```yaml
npm:
  select:
    - Nodes.CustomProperties.Infra_Logical_Location
    - IOSVersion
    - Nodes.CustomProperties.Infra_Location
```

**Groups:** Nornir groups are created using the *groups* list of dictionaries with group membership based around the device attribute *MachineTypes*.

-Group: Name of the group
-type: Host data dict to represent the device type for this group (router, switch, etc), it replaces MachineType
-filter: A list of upto two filter objects (and logic) to match against SELECT 'MachineTypes'
-scrapli: Optional 3rd connection driver added to connection_options (platform)
-netmiko: Optional  3rd connection driver added to connection_options (platform)
-napalm: Optional  3rd connection driver added to connection_options (platform)

This will create groups for *ios*, *iosxe*, *nxos*, *wlc*, *asa*, *wlc* and *checkpoint* with a custom *type* host_var and the *platform* set to  define the 3rd party connection driver.

```yaml
groups:
  - group: ios
    type: switch
    filter: [Catalyst, C9500]
    naplam: ios
    netmiko: cisco_ios
    scrapli: cisco_iosxe
  - group: iosxe
    type: router
    filter: [ASR, CSR]
    naplam: ios
    netmiko: cisco_iosxe
    scrapli: cisco_iosxe
  - group: nxos
    type: dc_switch
    filter: [Nexus]
    naplam: nxos_ssh
    netmiko: cisco_nxos_ssh
    scrapli: cisco_nxos
  - group: wlc
    type: wifi_controller
    filter: [WLC]
    netmiko: cisco_wlc_ssh
  - group: asa
    type: firewall
    filter: [ASA]
    netmiko: cisco_asa_ssh,
  - group: checkpoint
    type: firewall
    filter: [Checkpoint]
    netmiko: checkpoint_gaia_ssh
 ```

## Installation and Prerequisites

Create your project, clone *nornir_orion* to the root of it and install the dependencies (mainly *nornir*, *orionsdk* and *rich*)/

```python
mkdir my_new_project
cd my_new_project
git clone https://github.com/sjhloco/nornir_orion.git
python -m venv ~/venv/new_project
source ~/venv/new_project/bin/activate
pip install -r nornir_orion/requirements.txt
```

## Using the Inventory

Below is a bare minimum of what is required to use orion as your inventory with runtime filtering. The only mandatory argument is the inventory settings which holds all the NPM details, will be prompted for passwords at runtime (if not set in inventory settings).

```python
from nornir_orion import orion_inv
nr = orion_inv.main("inv_settings.yml")
```

It is also possible to add the True argument which will use a static inventory instead of Orion (looks for *hosts.yml* and *groups.yml* in */inventory*).

```python
nr = orion_inv.main("inv_settings.yml", True)
```

### Runtime flags

The following flags can be used to override the npm and device username specified in the inventory settings (*inv_settings.yml*).

| flag           | Description |
| -------------- | ----------- |
| -nu or --npm_user | Overrides the value set in *npm.user* variable |
| -du or --device_user | Overrides the value set in *device.user* variable |

Runtime filters (flags) can be used in any combination to filter the hosts in the inventory that the tasks will be run against. Filters are sequential so the ordering is of importance. For example, the second filter will only be run against hosts that have already matched the first filter. Words separate by special characters or whitespaces will need to be encased in brackets.

| filter            | method   | Options |
| ------------------| -------- | ------- |
| -h or --hostname  | contains | * |
| -g or --group     | any      | ios, iosxe, nxos, wlc, asa (includes ftd), checkpoint |
| -l or --location  | any      | DC1, DC2, DCI (Overlay), ET, FG |
| -ll or --logical  | any      | WAN, WAN Edge, Core, Access, Services |
| -t  or --type     | any      | firewall, router, dc_switch, switch, wifi_controller |
| -v  or --version  | contains | * |

These additional flags can be used to help with the forming of filters by displaying what hosts the filtered inventory will hold. If either of these are defined no actual inventory object is returned, so it prints the inventory and exits.

| flag | Description |
| ---- | ----------- |
| -s or --show | Prints all the hosts within the inventory |
| -sd or --show_detail | Prints all the hosts within the inventory including their host_vars |


All hosts in groups *ios*\
`python example_basic.py -s -g ios`

All hosts in groups *ios* or *iosxe* that have *WAN* in their name\
`python example_basic.py -s -g ios iosxe -n WAN`

All hosts (including host_vars) in group *ios* running version *16.9.6* at locations *DC* or *AZ*\
`python example_basic.py -sd -g iosxe -v "16.9.6" -l DC AZ`

!!!!! ADD Video !!!!!

### Adding additional flags to the Inventory

This other example (*example_adv.py*) takes it one step further and adds additional runtime flags to those used by *nornir_orion*. The new class (*NewProject*) gathers the arguments (flags) from *OrionInventory.add_arg_parser* and adds any additional arguments.

```python
from nornir_orion import orion_inv

no_orion = True

class NewProject:
    def __init__(self, orion):
        self.orion = orion

    def add_arg_parser(self):
        args = self.orion.add_arg_parser()
        args.add_argument("-f", "--filename", help="Name of the Yaml file containing ACL variables")
        args.add_argument("-a", "--apply", action="store_false", help="Apply changes to devices, by default only 'dry run'")
        return args
```

*OrionInventory.main* is copied from *nornir_orion* but instead of calling *OrionInventory.add_arg_parser* directly it calls *NewProject.add_arg_parser*. The rest of the function is the same, with all the arguments parsed and the inventory generated.

```python
def main(inv_settings: str, no_orion: bool = no_orion):
    orion = orion_inv.OrionInventory()
    my_project = NewProject(orion)
    inv_validate = orion_inv.LoadValInventorySettings()

    tmp_args = my_project.add_arg_parser()
    args = vars(tmp_args.parse_args())
    inv_settings = inv_validate.load_inv_settings(args, inv_settings)

    if no_orion == False:
        orion.test_npm_creds(inv_settings["npm"])
        nr_inv = orion.load_inventory(inv_settings["npm"], inv_settings["groups"])
    elif no_orion == True:
        nr_inv = orion.load_static_inventory("inventory/hosts.yml", "inventory/groups.yml")

    nr_inv = orion.filter_inventory(args, nr_inv)
    nr_inv = orion.inventory_defaults(nr_inv, inv_settings["device"])
    return nr_inv

if __name__ == "__main__":
    main("inv_settings.yml")
```
