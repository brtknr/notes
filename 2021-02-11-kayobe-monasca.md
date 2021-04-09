---
date: 11 February 2021
---

# Working with Magnum via Kayobe

To build and deploy new monasca-api image:

    kayobe overcloud container image build monasca-api -e kolla_install_type=source --push
    kayobe overcloud container image build monasca-persister -e kolla_install_type=source --push
    kayobe overcloud service reconfigure -kt monasca

To grab password from passwords file:

    ansible-vault view $KAYOBE_CONFIG_PATH/kolla/passwords.yml --vault-password-file ~/vault-pass | grep '^database_password'
    export KOLLA_DB_ADMIN_PASSWORD=xxxx