# Neos.Media-multiuser

Currently, the Neos Media module offers no easily feasible path for multi-user or multi-site 
support.

- **multi-user support**: even on a single site there can be a demand to have users of different
departments may not interfere with assets of other departments. Here you'll need multi-user 
support for assets.


- **multi-site support**: to avoid users for different sites access or utilize assets of 
not-their-site it is essential to have multi-site support for assets. Again, this is 
achievable utilizing the Neos.Media multiuser mods.

All this is possible utilizing NEOS' powerful privilege system.

Asset collection names can hold any characters (up to 255). With an agreement of
specific asset collection titles (see below **Working with specific asset collection titles**)
in interaction with privilege method `titleStartsWith` for asset collection titles
multi-user and multi-site support in the Neos Media module is achieved.

### Configuration example

In this example the agreement of specific asset collection titles follows the habit of path names
using the slash (`/`) character to separate parts of asset collection titles:
```
hierarchy              asset collection title
---------------------  -------------------------
apples                 apples
 + green               apples/green
 | + granny smith      apples/green/granny smith
 + red                 apples/red
   + fuji              apples/red/fuji
pears                  pears
 + green               pears/green
 | + french butter     pears/green/french butter
 + yellow              pears/yellow
   + bosc              pears/yellow/bosc
   + asian             pears/yellow/asian
```
The separation character can be configured in the Neos Media Browser module file `Settings.yaml`
(see below). If configured to anything else than empty (`''`) the asset collection title 
hierarchy is reflected in the asset collection's display:

![image](https://github.com/tdausner/Neos.Media-multiuser/assets/19776511/e4953b01-06c6-4315-a806-aac303817f52)

In a file `Policy.yaml` (in some `Configuration` folder) you may define corresponding roles:
```yaml
privilegeTargets:
  'Neos\Media\Security\Authorization\Privilege\ReadAssetCollectionPrivilege':
    'Apples.Asset:Collection':
      label: 'All apples asset'
      matcher: 'titleStartsWith("apples")'

    'Pears.Asset:Collection':
      label: 'All pears asset'
      matcher: 'titleStartsWith("pears")'

  # copy from Packages/Application/Neos.Media.Browser/Configuration/Policy.yaml
  'Neos\Flow\Security\Authorization\Privilege\Method\MethodPrivilege':
    'Neos.Media.Browser:ManageAssetCollections':
      label: Allowed to manage asset collections
      matcher: 'method(Neos\Media\Browser\Controller\(Asset|Image)Controller->(createAssetCollection|editAssetCollection|updateAssetCollection|deleteAssetCollection)Action()) || method(Neos\Media\Browser\Controller\AssetCollectionController->(create|edit|update|delete)Action())'

roles:

  # don't forget to give the Neos Administrator role access
  'Neos.Neos:Administrator':
    privileges:
      -
        privilegeTarget: 'Apples.Asset:Collection'
        permission: GRANT
      -
        privilegeTarget: 'Pears.Asset:Collection'
        permission: GRANT

  'Apples.Asset:Manager':
    label: 'Apple asset manager'
    description: 'Asset manager for assets in "apples" asset collections'
    parentRoles: ['Neos.Neos:Editor']
    privileges:
      -
        privilegeTarget: 'Apples.Asset:Collection'
        permission: GRANT
      -
        privilegeTarget: 'Neos.Media.Browser:ManageAssetCollections'
        permission: GRANT

  'Pears.Asset:Manager':
    label: 'Pears asset manager'
    description: 'Asset manager for assets in "pears" asset collections'
    parentRoles: ['Neos.Neos:Editor']
    privileges:
      -
        privilegeTarget: 'Pears.Asset:Collection'
        permission: GRANT
      -
        privilegeTarget: 'Neos.Media.Browser:ManageAssetCollections'
        permission: GRANT

```

## Installation
```shell
composer require tdausner/neos-media-multiuser
```
## Implementation

The privilege methods `titleStartsWith`, `titleEndsWith` and `titleContains`
are available for asset titles and do operate identical on asset collection titles
when copied to the php source file 
`Packages/Application/Neos.Media/Classes/Security/Authorization/Privilege/Doctrine/AssetCollectionConditionGenerator.php`.

Some other Media interface features screw up the separation of assets for different
sites:
- opportunity to select "All" collections
    - here you would see all assets of all sites
- Tags
    - it is hard to manage tags to differentiate assets for separate sites

The Neos Media module offers som feature configurations (file
`Packages/Neos/Neos.Media.Browser/Configuration/Settings.yaml`)
which are extended

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
        # set to non-empty character to enable (for example '/')
        collectionTree:
          separator: '/'
```
The collection tree separator is evaluated in file 
`Packages/Application/Neos.Media.Browser/Classes/Controller/AssetController.php` and in the media
index file `Packages/Application/Neos.Media.Browser/Resources/Private/Templates/Asset/Index.html`.

Some style adaptions for tree view are done in file 
`Packages/Application/Neos.Media.Browser/Resources/Public/Styles/MediaBrowser.css`.

For multi-site setup see [Neos PR#4426](https://github.com/neos/neos-development-collection/pull/4426) 
