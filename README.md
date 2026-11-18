# blog-bind-catalog-zones

Companion code for the blog post
[Set Up Bind DNS for Catalog Zones](https://blog.bsdserver.nl/how-to-set-up-bind-dns-for-catalog-zones/).

It builds the post's setup: a master (`10.71.0.176`) that serves the catalog zone
`catalog.example.com` (format version 2) and adds member zones with `rndc addzone`, and a
slave (`10.71.0.177`) that follows the catalog: every member listed there is created on the
slave automatically and stored in `cat-zones/`, and removed again when it leaves the catalog.

For every member zone the playbook does what the post does by hand:

1. writes the zone file on the master and runs `rndc addzone <zone> '{type master; file "<zone>";};'`;
2. adds `<sha1>.zones PTR <zone>.` to the catalog, with the same SHA-1 label as `nzf.py` and
   `openssl sha1` in the post (`47ac1a4d93b61fffdb4762c18c9e7d1a6b046d33` for `example.org`),
   bumps the catalog serial and runs `rndc reload catalog.example.com`.

Optional metadata from the post's last section is supported per zone: `masters` becomes
`masters.ext.<sha1>.zones A`, `allow_transfer` becomes `allow-transfer.ext.<sha1>.zones APL`.

## Differences from the post

- The post uses FreeBSD's `/etc/namedb` as the BIND directory. The role uses the
  distribution's own directory: `/var/cache/bind` on Debian/Ubuntu (AppArmor only lets
  `named` write there and in `/var/lib/bind`), `/var/named` on RHEL. File names inside it
  (`catalog.example.com.db`, `example.org`, `cat-zones/`) are the post's.
- The catalog serial is a Unix timestamp, so it only ever goes up; it changes only when the
  member list changes.

## What is in here

| Path | Purpose |
|------|---------|
| `terraform/` | Two Ubuntu 24.04 VMs on Proxmox VE (bpg/proxmox provider, cloud-init), outputs an Ansible inventory |
| `ansible/site.yml` | One playbook for the whole setup |
| `ansible/roles/bind` | Installs BIND and writes `named.conf` with includes |
| `ansible/roles/bind_catz` | Master and slave options, catalog zone, member zones |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VMs)
- ansible-core >= 2.15 (tested with 2.20.1)
- Target hosts: Ubuntu 24.04 (tested, BIND 9.18) or RHEL/Rocky 9 (handled, not lab-tested)

## Quick start

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init && terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

cd ../ansible                                   # or: cp inventory.example.yml inventory.yml
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml

dig +short @10.71.0.177 example.org SOA         # served by the slave, from the catalog
ssh 10.71.0.177 ls -l /var/cache/bind/cat-zones
```

To add a zone, append it to `bind_catz_members` and run the playbook again. To remove one,
set `state: absent` on it: the master drops it from the catalog (the slave deletes its
copy) and then removes it with `rndc delzone`.

## Variables

Set in `ansible/group_vars/all/vars.yml` (defaults in `roles/bind_catz/defaults/main.yml`):

| Variable | Default | Meaning |
|----------|---------|---------|
| `bind_catz_master_address` | `10.71.0.176` | Master; the slave's `masters` and `default-masters` |
| `bind_catz_slave_addresses` | `["10.71.0.177"]` | Slaves allowed to transfer the catalog, and notified |
| `bind_catz_catalog` | `catalog.example.com` | Catalog zone name |
| `bind_catz_zone_directory` | `cat-zones` | Where the slave stores member zones (below the BIND directory) |
| `bind_catz_members` | `example.org` | Member zones: `name`, `file`, optional `state`, `masters`, `allow_transfer` |

Hosts in `catz_master` become the master, hosts in `catz_slaves` slaves. This setup has no
secrets, so `group_vars/all/vault.yml.example` is empty.

## Tested

`site.yml` was run against fresh Ubuntu 24.04 systemd containers (BIND 9.18.39); the second
run reports `changed=0`. Checks afterwards: the catalog has the post's label for
`example.org`, the slave answers for `example.org` with `aa` and has
`cat-zones/__catz___default_catalog.example.com_example.org.db`; with `state: absent` the
slave logs `catz: deleting zone 'example.org'` and answers REFUSED; re-adding it with
`allow_transfer` metadata plus a second zone (`example.net`) brings both back on the slave,
and AXFR of `example.org` from an address outside the APL fails while `example.net` transfers.
ansible-lint (production profile), yamllint, `terraform validate` and tflint are clean. The
Terraform code has not been applied against a real Proxmox cluster.

## License

MIT
