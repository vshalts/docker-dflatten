dflatten is simple script that allow to flatten docker image created with aufs.
===============

The main difference from another prototypes that this script does not try to cheat by emulating aufs behavior (doesn't work with .wh. files directly).
Instead it force aufs to do its job and uses aufs in order to create difference between images.

dflatten will not remove/change old layers, but create new flatten image directly under parent image.

Script reads data from /var/lib/docker and mount aufs, so root privileges is required.
But script doesn't do any modifications in /var/lib/docker, only read operations.
All intermediate data created inside /tmp folder.

Requirements
===============
Script depend on jq (http://stedolan.github.io/jq/), rsync and aufs
On debian and Ubuntu jq available as standard package:
sudo apt-get install -y jq

How to use
===============
The simplest way to run script:
```bash
> sudo dflatten -o vshalts/general
```
where `vshalts/general` is should be replaced with your image name.

### Real example:

<pre>
docker images -tree
Warning: '-tree' is deprecated, it will be removed soon. See usage.
└─511136ea3c5a Virtual Size: 0 B
  └─6170bb7b0ad1 Virtual Size: 0 B
    └─9cd978db300e Virtual Size: 204.4 MB
      └─a6d44c263269 Virtual Size: 204.4 MB
        └─13## Heading ##e42d0c2a51 Virtual Size: 204.4 MB
          └─846e143f3fab Virtual Size: 204.4 MB
            └─10ebd1d649cb Virtual Size: 204.4 MB
              └─5db917407faa Virtual Size: 345.8 MB
                └─745d3ac92697 Virtual Size: 345.8 MB Tags: phusion/baseimage:0.9.9
                  ├─22865bb012e7 Virtual Size: <b>345.8</b> MB Tags: <b>vshalts/general_flatten:latest</b>
                  └─c18b95098dea Virtual Size: 345.8 MB
                    └─3f420d29853d Virtual Size: 345.8 MB
                      └─3ea7162b6a47 Virtual Size: 348.6 MB
                        └─8ea00a46aa21 Virtual Size: 348.6 MB
                          └─b6a4f88b4d73 Virtual Size: 471.8 MB
                            └─86a85dc75b6e Virtual Size: 475.5 MB
                              └─c2a2677c746d Virtual Size: 513.9 MB
                                └─3d2c2ec3ece3 Virtual Size: 513.9 MB
                                  └─b06ef8c3fc3a Virtual Size: <b>513.9</b> MB Tags: <b>vshalts/general:latest</b>
</pre>
Here flatten image 22865bb012e7 was created and size was reduced from 513.9 down to 345.8.


By default script will add '_flatten' to specified image name. But you can overwrite this behavior and define required target image name with -t option:

```bash
> sudo dflatten -o vshalts/general -t vshalts/new_image
```

You can also use tags:
```bash
> sudo dflatten -o vshalts/general:1.0.0 -t vshalts/new_image:2.0.0
```

How does it work?
===============

Script will mount layers of parent image in read-only mode and then add another empty writable layer on top.
After that it will rsync changes from original image to this cake. This will force aufs to put all changes in writable layer.
At last step it create tar archive which contain this writable layer and then load image with `docker load`.

Image id of generated flatten image is reversed ID of original image. This makes generated ID stable.
Script will use this fact to implement simple caching behavior (will not rebuild flatten image if image with required ID already exist).


#### Cons:
- Script correctly flatten any complex layers (including case where original image remove files from parent image)
- Simple. You can easily modify it for your purpose.
- Very fast
- Support caching
- Quite safe (it doesn't modify /var/lib/docker directly and doesn't remove/change existing images)

#### Pros:
- only support aufs
- root privileges required (for mount and read access to /var/lib/docker)