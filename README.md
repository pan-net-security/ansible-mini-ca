# ansible-mini-ca

This role signs a (set of )certificate(s) for every host in the play, using variable-configured CA.

If the CA (key/crt) is empty it will generate one itself.

## Requirements

* openssl on every play host + localhost
* writable temporary directory on localhost

## Read this

* Playing the role with default vars will *not* persist your CA. Setting `mini_ca_ca_cleanup` to `false` will keep temporary files in `mini_ca_temp_path` (defaults to `/tmp`)
* If there are pre-existing certificates on remote hosts, signed by a different CA, both key/crt will be deleted and recreated
* If you set `mini_ca_ca_key` to some key and leave `mini_ca_ca_cert` empty, it will work
* If you set `mini_ca_ca_cert` to some crt and leave `mini_ca_ca_key` empty, it will not work (and fail badly)
* Consider using `any_errors_fatal: true` for your play if your app relies on all certificates being issued.

## Role Variables

* `mini_ca_ca_key` - private key of the CA in PEM format. Defaults to `''`, causing role to generate one. If you need it to persist, save the output yourself.
* `mini_ca_ca_cert` - CA cert in PEM format. Defaults to `''`, causing role to generate one. If you need it to persist, save the output yourself.
* `mini_ca_temp_path` - writable directory for storing temp files
* `mini_ca_ca_cn` - CN for your shiny metal CA

Check for all vars in [defaults/main.yml](defaults/main.yml), it's pretty much self-explaining.

## Saving the output

If your CA was created in a play, you should save the output value of `mini_ca_ca_key` and `mini_ca_ca_cert`, either with

```
- hosts: myhosts
  tasks:
    - name: Save CA secrets
      copy:
        content: "{{ item.content }}"
        path: "{{ item.where }}"
      with_items:
        - content: "{{ mini_ca_ca_key }}"
          where: "/tmp/ca-key.pem"
        - content: "{{ mini_ca_ca_cert }}"
          where: "/tmp/ca-crt.pem"
      run_once: true
      delegate_to: localhost
```

or with [ansible-save-secrets](https://github.com/pan-net-security/ansible-save-secrets/):

```
- hosts: myhosts
  become: false
  vars:
    - secret_store: "vault"
    - vault_mount: "secret"
    - vault_path: "myca"
    - vars_stored:
      - var: "mini_ca_ca_key"
        key: "key"
      - var: "mini_ca_ca_cert"
        key: "crt"
  roles:
    - ansible-save-secrets
```

## Example Playbook

For a very simple one-cert scenario with `CN={{ ansible_fqdn }}` it's sufficient to call role with defaults.

### Single cert

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: all
      roles:
         - { role: ansible-mini-ca, mini_ca_ca_cn: "MyDamnGoodPrivateCA" }

Result of this play is:

  * your CA private key in `mini_ca_ca_key`
  * your CA certificate in `mini_ca_ca_cert`
  * `/etc/ssl/certs/{{ inventory_hostname }}.pem` on every play host, signed by "MyDamnGoodPrivateCA"
  * `/etc/ssl/private/{{ inventory_hostname }}.pem` on every play host, signed by "MyDamnGoodPrivateCA"

### Multiple certs

If you need more than one certificate, fill in `mini_ca_certs_to_sign` list with following attributes:

* `cn` (mandatory) - certificate common name
* `cert_path` (mandatory) - path to certificate
* `key_path` (mandatory) - path to privkey
* `csr_path` (mandatory) - path to CSR (temporary)
* `subject` - certificate subject
* `owner`, `group`, `mode` - file owners/permissions for the cert

```
    - hosts: all
        vars:
          mini_ca_certs_to_sign:
            - cn: "{{ ansible_fqdn }}"
              cert_path: "/etc/ssl/certs/{{ ansible_fqdn }}.pem"
              key_path: "/etc/ssl/private/{{ ansible_fqdn }}.pem"
              csr_path: "/tmp/{{ ansible_fqdn }}.csr"
              subject: "/C=CZ/ST=Slovakia/L=Bratislava/O=DTPN/OU=infra-security/CN={{ ansible_fqdn }}/ }}"
            - cn: "something-else.pan-net.eu"
              cert_path: "/etc/ssl/certs/webserver.pem"
              key_path: "/etc/ssl/private/webserver.pem"
              csr_path: "/tmp/webserver.csr"
              subject: "/CN=something-else.pan-net.eu/ }}"
          mini_ca_ca_cn: "MyDamnPerfectCA"
      roles:
         - ansible-mini-ca
```

## License

GPL

## Author Information

Michal Medvecky <michal@medvecky.net>
