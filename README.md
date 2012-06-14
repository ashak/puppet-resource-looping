puppet-resource-looping
=======================

I often find myself having an array of something that I then wish to loop
though within a puppet manifest to create resources. I'm sure we all know that
Puppet does not contain loops unless you abuse a define for the purpose.
Even then it's not always possible to achieve what in your head is so simple
and yet within your Puppet manifest is almost impossible.

A quick example:

Imagine i'm working with some firewall rules for a service. I have an array
containing an element for each network that's allowed to talk to this service:

    $allowed_networks = ['10.0.0.0/8', '192.168.0.0/24']

I know what my firewall definition should look like..
    firewall { '100 allow <network> to apache on ports 80':
      'proto'  => 'tcp',
      'action' => 'accept',
      'dport'  => 80,
      'source' => <network>,
    }

Now... if I could just do this:
    $allowed_networks.each do |network|
      firewall {....
      }
    end

But I can't. Yes, I could write a define like this:
    define my_apache_firewall_wrapper() {
      firewall { "100 allow ${name} to apache on ports 80":
        'proto'  => 'tcp',
        'action' => 'accept',
        'dport'  => 80,
        'source' => $name,
      }
    }

And use it like this:
  my_apache_firewall_wrapper{ $allowed_networks: }

And it would be called once for each element of $allowed_networks, win.. no?

Why?
* If I want to create firewall rules for something else, I need to create a new
  define. Yes... I could allow some extra parameters be passed to the define
  above, but you soon get to the stage where you need to do something that you
  just can't do.
* Abusing defines in this way is horrible, unreadable and is not reusable. You
  end up with lots of defines, all with slightly obscure names, all doing the
  same sort of thing in slightly different ways.

So i've written some simple functions to perform loop like tasks within puppet

Enter the function: create_resources_hash_from
create_resources_hash_from(
  <a formatted string>,
  <an array to loop on>,
  <a hash to template your resource>,
  [an optional array of parameters to add to each resource])

So lets start with our allowed_hosts array again:
    $allowed_networks = ['10.0.0.0/8', '192.168.0.0/24']

Then a formatting string for the name of your resource:
    $resource_name = "100 allow %s to apache on ports 80"

A hash laying out a template for each resource that you wish to create:
    $my_resource_hash = {
      'proto'  => 'tcp',
      'action' => 'accept',
      'dport'  => 80
    }

And a list of parameters that you wish to have added to your resource based
on the thing that you're looping on:
    $dynamic_parameters = ['source']

    $my_resources_hash = create_resources_hash_from($resource_name, $allowed_hosts, $my_resource_hash, $dynamic_parameters)

Then simply pass your new hash and the name of the resource that you wish to
create to create_resources:
    create_resources(firewall, $my_resources_hash)

Much nicer.. at least IMHO :-)
