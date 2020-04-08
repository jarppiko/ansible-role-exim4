# Ansible Role: Exim

Installs exim on RedHat/CentOS or Debian/Ubuntu.

## Requirements

If you're using this as an SMTP relay server, you will need to do that on your own, and open TCP port 25 in your server firewall.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

    exim_config_file: /etc/exim4/exim4.conf

The path to the Exim `main.cf` configuration file.

    exim_service_state: started
    exim_service_enabled: yes

The state in which the Exim service should be after this role runs, and whether to enable the service on startup.

    exim_inet_interfaces: localhost
    exim_inet_protocols: all

Options for values `inet_interfaces` and `inet_protocols` in the `main.cf` file.

## Dependencies

None.

## Example Playbook

    - hosts: all
      roles:
        - rivimey.exim

## License

MIT / BSD

## Author Information

This role was created in 2018 by [Ruth Ivimey-Cook](https://www.ivimey.org/).
