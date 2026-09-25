---
layout: post
title: "Getting a VS Code Server running on EC2" 
date: 2023-07-16 10:27:00 +0100
comments: false
categories: Cloud
---

As part of the preparations for the [workshop on container security](https://www.steelcon.info/the-event/workshops/)  that myself and [Iain Smart](https://twitter.com/smarticu5/) ran at this year's [Steelcon](https://www.steelcon.info/), there were some concerns that our standard option of SSH access might be blocked by the venue's Wi-Fi, so a backup plan was in order. As a result, we were looking into how to provide a browser based terminal for students running on a host in AWS EC2. 

After looking at various options, we decided to see if we could get [code-server](https://github.com/coder/code-server) working. It's a really nice project that provides a hosted version of VS Code, which gives you file exploring/editing, a terminal and port forwarding for local applications, all in a browser.

After the conference, I decided to extract the config into a stand-alone set of Ansible tasks, and put it into my [Container Security Workstation](https://github.com/raesene/container_sec_workstation) repo. You can see the overall playbook [here](https://github.com/raesene/container_sec_workstation/blob/main/ec2_container_workstation.yml) but we'll go through the key parts in this post.

There were a couple of interesting technical aspects to getting it all working, which I thought I'd write-up here, in case it's of use to other people!

## Setting up an EC2 instance with Ansible

This is relatively straight-forward, but with one caveat, that Ansible changed the syntax of this, so if you have an older version of ansible this may not work. First up install the AWS ansible galaxy role `ansible-galaxy collection install amazon.aws` then have a block like this to setup the EC2 and wait for it to be available

```yaml
- name: start an instance with a public IP address
  amazon.aws.ec2_instance:
    name: "Container Security Workstation"
    key_name: "{% raw %}{{ key_pair }{% endraw %}}"
    vpc_subnet_id: "{% raw %}{{ subnet_id }}{% endraw %}"
    region: "{% raw %}{{ region }}{% endraw %}"
    instance_type: "{% raw %}{{ instance_type }}{% endraw %}"
    security_group: "{% raw %}{{ security_group }}{% endraw %}"
    network:
      assign_public_ip: true
    image_id: "{% raw %}{{ ami_id }}{% endraw %}"
    wait_timeout: 60
    state: running
  register: ec2

- name: Add all instance public IPs to host group
  add_host: hostname={% raw %}{{ item.public_ip_address }}{% endraw %} groups=ec2host
  loop: "{% raw %}{{ ec2.instances }}{% endraw %}"
  
- name: Wait for SSH to be available
  delegate_to: "{% raw %}{{ item.public_ip_address }}{% endraw %}"
  wait_for_connection:
    delay: 20
    timeout: 60
  with_items: "{% raw %}{{ ec2.instances }}{% endraw %}"
```

A couple of key points to note for this. you'll need a valid API connection to AWS with enough rights to create an EC2 instance. You'll then need to have the information for the various variables here

 - `key_pair` - The name of an SSH key pair in your AWS account to use for access to the host
 - `subnet_id` - The ID of the subnet to place the EC2 in
 - `region` - The region to use
 - `instance_type` - The EC2 instance type to use
 - `security_group` - A security group that exists in your AWS account which allows at least 22/TCP, probably also 443/TCP for Caddy.
 - `ami_id` - the AMI ID to use for the host. In my case I use an ubuntu:22.04 based AMI.

## Code Server Install and Config

The basic installation of Code server is pretty straight-forward. They provide a deb package, so we can just download and install that :-

```yaml
  - name: Install Code Server
    get_url:
      url: https://github.com/coder/code-server/releases/download/v4.14.0/code-server_4.14.0_amd64.deb
      dest: /tmp/code-server.deb

  - name: Install Code Server
    apt:
      deb: /tmp/code-server.deb
      state: present
  
  - name: start code server
    systemd:
      name: code-server@ubuntu
      state: started
      enabled: yes
```

After that there's a couple of configuration changes to make, first I wanted to move the port from `8080` to `18080` as I'll often use `8080` for other things. Using Ansible's `lineinfile` was a good way to do that, and pointing it at the default config file location in the user's home directory. As we're using an Ubuntu EC2 instance here, that'll be `/home/ubuntu`.

```yaml
  - name: Change port to 18080 in the code-server config file
    lineinfile:
      path: /home/ubuntu/.config/code-server/config.yaml
      regexp: '^bind-addr:'
      line: 'bind-addr: 127.0.0.1:18080'
```

You might notice that we're still listening on `127.0.0.1` here, but we'll get to making this accessible remotely in a bit!

Next up we want to change the password from the default value, of course. Code Server has different [authentication options](https://coder.com/docs/code-server/latest/guide#external-authentication) available, but for this purpose, a static password (assuming it's suitably strong) should be fine.

```yaml
  - name: Change password to the value of code_password in the code-server config file
    lineinfile:
      path: /home/ubuntu/.config/code-server/config.yaml
      regexp: '^password:'
      line: 'password: {% raw %}{{code_password}}{% endraw %}'
```

This section just sets the password to whatever is held for the ansible var `code_password`. 

Lastly for this section, we want to re-start the server, so our configuration changes take effect.

```yaml
  - name: restart code server
    systemd:
      name: code-server@ubuntu
      state: restarted
      enabled: yes
```

## Making it available remotely - Caddy!

I've mentioned before about [how cool Caddy is](https://raesene.github.io/blog/2023/01/21/Fun-with-Caddy-SSRF-Testing/) for a variety of reasons, and we can make use of it here to expose the Code server over TLS. As a pre-requisite, this section uses an Ansible galaxy role, which can be installed with `ansible-galaxy collection install maxhoesel.caddy`. If you're happy enough with SSH port-forwarding, that would be another option here.

We want to change the configuration of Caddy so that it'll provide a reverse proxy from `127.0.0.1:18080` to `0.0.0.0:443` and set-up TLS

```yaml
 - name: Setup the Caddy server
    include_role: 
      name: maxhoesel.caddy.caddy_server
    vars:
      caddy_config_mode: "Caddyfile"
      caddy_caddyfile: |
        :443 {
          tls internal {
            on_demand
          }
          reverse_proxy :18080
        }
        
  - name: Restart Caddy
    systemd:
      name: caddy
      state: restarted
      enabled: yes
```

## Extra Credit - Valid TLS cert

At this point you've got a configuration that'll work, but the certificate won't be trusted by the browser, which isn't ideal (if only that you'll need to click through a warning when you get to it). We can get TLS certificates issued on demand using [Lets encrypt](https://letsencrypt.org/) and Caddy which has a very neat trick of provisioning these on the fly, we just need a valid DNS record for the host.

Of course you could do this manually after set-up, but it'd be neat to have it provisioned automatically. Here we just need a DNS provider that's got an API and, ideally, Ansible integration. Fortunately the DNS provider I use [DNSimple](https://dnsimple.com) has both of these things!

First up we need to register our DNS record. This process will vary depending on your provider but the general concepts will likely remain. You'll need an API key for the provider and a domain to host at. The task below sets up a host called `csw` and a domain specified as the var `dns_domain` and sets the `A` record to point to `inventory_hostname` which should be the external IP address of the EC2 we've started.

```yaml
  - name: Authenticate to DNSimple & Create Record
    community.general.dnsimple:
      account_email: "{% raw %}{{ dnsimple_account_email }}{% endraw %}"
      account_api_token: "{% raw %}{{ dnsimple_api_key }}{% endraw %}"
      domain: "{% raw %}{{ dns_domain }}{% endraw %}"
      type: A
      # Change as needed
      record: csw
      solo: yes
      ttl: 360
      value: "{% raw %}{{inventory_hostname}}{% endraw %}"
      state: present
    delegate_to: localhost
    register: dns_record
```

Now we've got a valid DNS record, we just need to re-write the Caddyfile that we're using so that Caddy will provision a cert for us on access. These tasks remove the old Caddyfile, add a new one specifying our host and domain and setting up TLS and then re-start Caddy.

```yaml
  - name: remove old Caddy Config
    file:
      path: /etc/caddy/Caddyfile
      state: absent

  - name: Add caddyfile contents
    copy:
      dest: /etc/caddy/Caddyfile
      content: |
        csw.{% raw %}{{dns_domain}}{% endraw %}:443 {
          tls {% raw %}{{dnsimple_account_email}}{% endraw %}
          reverse_proxy :18080
        }

  - name: Restart Caddy
    systemd:
      name: caddy
      state: restarted
      enabled: yes
```

If all works, once you've run your playbook, it should look something like this :-

![VS Code in browser]({{site.url }}/assets/media/csw-vscode.png)

## Conclusion

At the end of all this you should have a nice Web hosted development/testing environment with a terminal and port-forwarding, which could be handy for a number of reasons.