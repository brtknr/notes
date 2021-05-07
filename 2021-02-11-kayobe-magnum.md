---
date: 11 February 2021
---

# Working with Magnum via Kayobe

## Prerequisites

Make sure that `kolla_openstack_release` inside `etc/kayobe/kolla.yml` and `magnum_tag` inside `etc/kayobe/kolla/globals.yml` match.

## Building image

    kayobe overcloud container image build magnum --push

NOTE: When `-e kolla_install_type=source` was added to above, it didn't like the fact that `python 2.7` was being used under the hood when `setup.cfg` requires `python 3.6+` explicitly.

## Deploy

    kayobe overcloud service reconfigure -kt magnum