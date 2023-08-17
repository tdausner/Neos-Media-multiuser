# Neos.Media.Multiuser

Currently, the Neos Media module offers no easily feasible path for multi-user or multi-site 
support.

- **multi-user support**: even on a single site there can be a demand to have users of different
departments may not interfer with assets of departments. That's when you'll need multi-user 
support for assets.
- **multi-site support**: to avoid users for different sites access assets of not-their-site
it is essential to have multi-site support for assets. Again, this is achievable utilizing 
the Neos.Media.Multiuser mods.

All this is possible utilizing NEOS' powerful privilege system.

Asset collection names can hold any characters (up to 255). With an agreement of
specific asset collection titles (see below **Working with specific asset collection titles**)
in interaction with privilege methods `titleStartsWith`, `titleEndsWith` and
`titleContains` for asset collection titles it is possible to utilize the Neos Media
module for multi-site installations

The privilege methods `titleStartsWith`, `titleEndsWith` and `titleContains`
are available for asset titles and do operate identical on asset collection titles
when copied to the corresponding php source.

Some other Media interface features screw up the separation of assets for different
sites:
- opportunity to select "All" collections
    - here you would see all assets of all sites
- Tags
    - it is hard to manage tags to differentiate assets for separate sites

The Neos Media module offers som feature configurations (see file
`Packages/Neos/Neos.Media.Browser/Configuration/Settings.yaml` of this PR)
which easily can be extended

```yaml
Neos:
  Media:
    # ...
    Browser:
      # ...
      features:
        # ...
        
        # Show "all" collections
        showCollectionsAll:
          enable: false
        # Show tags
        showTags:
          enable: false
```

With these feature entries the Neos Media module can be customized as required for multi-site support.

**Review instructions**

- start with a fresh NEOS development installation (see https://github.com/neos/neos-development-collection#contributing)
    - `tds` is my development domain
    - shell command (adapted from https://discuss.neos.io/t/development-setup/504)
```shell
$ composer create-project neos/neos-development-distribution neos-dev.tds @dev --keep-vcs
```

- setup test sites

```shell
for s in First Second Third
do
  echo "0" | ./flow kickstart:site $s.Site $s
  ./flow site:import --package-key $s.Site
  ./flow domain:add ${s,,}-site ${s,,}.tds --scheme https
done
```

Retrieve the node names
```shell
$ ./flow site:list

 Name                 Node name                 Resources package               Status
----------------------------------------------------------------------------------------
 First                first-site                First.Site                      online
 Second               second-site               Second.Site                     online
 Third                third-site                Third.Site                      online
```

- Setup `Policy.yaml` for all three sites. Example file is
  `DistributionPackages/First.Site/Configuration/Policy.yaml`

```yaml
privilegeTargets:
  'Neos\Neos\Security\Authorization\Privilege\NodeTreePrivilege':

    'First.Site:NodeTreePrivilege':
      label: 'Access to First pages'
      matcher: 'isDescendantNodeOf("/sites/first-site")'

  'Neos\Media\Security\Authorization\Privilege\ReadAssetCollectionPrivilege':

    'First.Site:FirstAssetCollection':
      label: 'First site asset collections access'
      matcher: 'titleStartsWith("first")'

  # copy from Packages/Application/Neos.Media.Browser/Configuration/Policy.yaml
  'Neos\Flow\Security\Authorization\Privilege\Method\MethodPrivilege':
    'Neos.Media.Browser:ManageAssetCollections':
      label: Allowed to manage asset collections
      matcher: 'method(Neos\Media\Browser\Controller\(Asset|Image)Controller->(createAssetCollection|editAssetCollection|updateAssetCollection|deleteAssetCollection)Action()) || method(Neos\Media\Browser\Controller\AssetCollectionController->(create|edit|update|delete)Action())'

roles:

  'Neos.Neos:Administrator':
    privileges:
      -
        privilegeTarget: 'First.Site:NodeTreePrivilege'
        permission: GRANT
      -
        privilegeTarget: 'First.Site:FirstAssetCollection'
        permission: GRANT

  'First.Site:FirstEditor':
    label: 'First Editor'
    description: 'Edit and release of First pages'
    parentRoles: ['Neos.Neos:Editor']
    privileges:
      -
        privilegeTarget: 'First.Site:NodeTreePrivilege'
        permission: GRANT

  'First.Site:FirstAssets':
    label: 'First Asset access'
    description: 'Access to First assets'
    privileges:
      -
        privilegeTarget: 'First.Site:FirstAssetCollection'
        permission: GRANT
      -
        privilegeTarget: 'Neos.Media.Browser:ManageAssetCollections'
        permission: grant
```
Result (if `Policy.yaml` is set for all three sites)
```shell
$ ./flow security:listroles
+-----------------------------------------------+---------------------+
| Id                                            | Label               |
+-----------------------------------------------+---------------------+
| Neos.Neos:LivePublisher                       | Live publisher      |
| Neos.Neos:Administrator                       | Neos Administrator  |
| Neos.Neos:RestrictedEditor                    | Restricted Editor   |
| Neos.Neos:Editor                              | Editor              |
| Neos.Neos:UserManager                         | Neos User Manager   |
| Third.Site:ThirdEditor                        | Third Editor        |
| Third.Site:ThirdAssets                        | Third Asset access  |
| First.Site:FirstEditor                        | First Editor        |
| First.Site:FirstAssets                        | First Asset access  |
| Second.Site:SecondEditor                      | Second Editor       |
| Second.Site:SecondAssets                      | Second Asset access |
| Neos.Setup:SetupUser                          | SetupUser           |
| Flowpack.Neos.FrontendLogin:User              | User                |
| Flowpack.Neos.FrontendLogin:UserAdministrator | UserAdministrator   |
+-----------------------------------------------+---------------------+
```
- Add site users `first`, `second` and `third`. Passwords are set to user's names:

```shell
for s in First Second Third
do
  ./flow user:create --roles ${s}.Site:${s}Editor,${s}.Site:${s}Assets ${s,,} ${s,,} $s User
done
```

```shell
$ ./flow user:list
+---------------+-------+-------------------+-----------------------------------------------------------+--------+
| Name          | Email | Account(s)        | Role(s)                                                   | Active |
+---------------+-------+-------------------+-----------------------------------------------------------+--------+
| First User    |       | first             | First.Site:FirstEditor, First.Site:FirstAssets            | yes    |
| Second User   |       | second            | Second.Site:SecondEditor, Second.Site:SecondAssets        | yes    |
| Third User    |       | third             | Third.Site:ThirdEditor, Third.Site:ThirdAssets            | yes    |
| Administrator |       | admin@mydomain.de | Neos.Neos:Administrator                                   | yes    |
+---------------+-------+-------------------+-----------------------------------------------------------+--------+
```
- Log into NEOS as administrator and add some asset collections

![image](https://github.com/neos/neos-development-collection/assets/19776511/92017274-abe2-4f75-b9d1-59a4fe5f2c8d)

- Add some assets and assign each asset to exactly one of the collections

![image](https://github.com/neos/neos-development-collection/assets/19776511/0a3ccfd2-d8c4-4631-b18c-ee85d1bf7bb1)

- Set default asset collection at sites in Site Management

![image](https://github.com/neos/neos-development-collection/assets/19776511/b747c6e8-3729-46dc-a832-ae9ca872a92f)

Repeat this step for sites `Second` and `Third`.

- The domains of the sites created are
```shell
$ flow domain:list
+-------------+---------------------------+--------+
| Node name   | Domain (Scheme/Host/Port) | State  |
+-------------+---------------------------+--------+
| first-site  | https://first.tds         | active |
| third-site  | https://third.tds         | active |
| second-site | https://second.tds        | active |
+-------------+---------------------------+--------+
```
It is mandatory to utilize these domains to access the Neos backends for operation of the site specific default asset collections.

- Log into Neos backend at `https://first.tds/neos` as user `first` (password `first`) and go to Media

![image](https://github.com/neos/neos-development-collection/assets/19776511/82d38267-0357-44d5-ab8b-cf35aba44a64)

- Log into Neos backend at `https://second.tds/neos` as user `second` (password `second`) and go to Media

![image](https://github.com/neos/neos-development-collection/assets/19776511/b45642b6-a5a9-4f5a-9562-1602d77b14b3)

### Working with specific asset collection titles

To work with specific asset collection titles set the feature parameter `collectionTree.separator` to
a character used to separate the different parts of the asset collection title (slash `/` for example)
in file `Packages/Neos/Neos.Media.Browser/Configuration/Settings.yaml` of this PR.
```yaml
Neos:
  Media:
    # ...
    Browser:
      # ...
      features:
        # ...
        
        # Show "all" collections
        showCollectionsAll:
          enable: false
        # Show tags
        showTags:
          enable: false
        # collection hierarchical tree view / separation character
        # set to '' (empty string) to disable
        collectionTree:
          separator: '/'
```
Log into Neos backend at `https://first.tds/neos` as user `first` (password `first`) and go to Media
and add two asset collections
- `first/one`
- `first/one/two`

results in a tree view (each asset collection has an asset assigned):

![image](https://github.com/neos/neos-development-collection/assets/19776511/ff3a8033-799b-4e61-b3b8-329bee36904e)

Clicking the "edit" pencil at the "Collections" shows that the "base" asset collection (**first**)
cannot be edited or deleted.

![image](https://github.com/neos/neos-development-collection/assets/19776511/e40c4c51-7260-41c5-9bd1-87d8052d900b)

Deleting one asset collection...

![image](https://github.com/neos/neos-development-collection/assets/19776511/0e5262ec-5734-4ba3-a49e-9ffaa38b9b52)

...moves the assets of the deleted asset collection to the "base" asset collection (**first**)

![image](https://github.com/neos/neos-development-collection/assets/19776511/dab94098-2771-46e1-8c3a-7802b020f36f)


**Checklist**

- [X] Code follows the PSR-2 coding style
- [ ] Tests have been created, run and adjusted as needed
- [ ] ~The PR is created against the [lowest maintained branch](https://www.neos.io/features/release-roadmap.html)~
- [x] Reviewer - PR Title is brief but complete and starts with `FEATURE|TASK|BUGFIX`
- [x] Reviewer - The first section explains the change briefly for change-logs
- [ ] ~Reviewer - Breaking Changes are marked with `!!!` and have upgrade-instructions~
