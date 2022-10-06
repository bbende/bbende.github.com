---
layout: post
title: "Apache NiFi 1.18.0 - Parameter Providers"
description: ""
author: Joe Gresock
category: "Development"
tags: [NiFi,Parameter Providers,Parameter Contexts]
---
{% include JB/setup %}

#### (Guest author: Joe Gresock)

A rundown of the new Parameter Providers extension point in Apache NiFi 1.18.0.

### Introduction

Apache NiFi 1.18.0 introduces a new extension point: the Parameter Provider.  This powerful feature allows automatic creation of
Parameter Contexts from external sources (e.g., file-based Kubernetes secrets, environment variables, HashiCorp Vault secrets engines).
Paired with Parameter Context inheritance, flow parameters are now more flexible than ever.  This post goes over some of the basics
of Parameter Providers, but first let's see what Parameter Providers do and don't do:

**What Parameter Providers Do:**
- Provide a new feature in the Controller Settings that let you generate Parameter Contexts from an external sources
- Let you manually keep your provided Parameter Contexts up to date with the external source by running a "Fetch Parameters" operation
- Provide an extension point for developing new custom Parameter Providers that can be deployed in a NAR.
- Provide a CLI command, `nifi fetch-params`, to fetch and apply parameters, allowing a scripted approach to keep Parameter Contexts up to date.

**What Parameter Providers Don't Do:**
- Parameter Providers don't pull parameters directly from the external source at the time of usage in the flow.  Parameter values are still stored (encrypted, if sensitive) inside the flow itself.  Think of Parameter Providers as a mechanism that automates the creation of Parameter Contexts and facilitates keeping them updated, not as something that replaces the framework mechanism to resolve parameter values during Processor/Controller Service execution.
- Parameter Providers don't have an automatic mechanism to refresh parameter values from the NiFi UI.  Fetching and applying parameters is a potentially disruptive operation, since it can involve stopping and starting large portions of the flow.  This kind of activity could be scripted using the CLI command `fetch-params`, but
should be done with the potential for flow disruption in mind.

Now, let's take a look at Parameter Providers and how they fit into the flow.

### Creating and Configuring a Parameter Provider

First, select Controller Settings from the top-right hamburger menu in NiFi:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/01-controller-settings-menu.png" class="img-responsive" width="30%" height="30%">

Then navigate to the new Parameter Providers tab and click the '+' button on the top-right of the Parameter Providers listing to add a new Parameter Provider.

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/02-create-parameter-provider.png" class="img-responsive">

For this example, select FileParameterProvider, and then Add.  The Parameter Provider is created, and now you can click the Pencil button to edit it.

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/03-configure-parameter-provider.png" class="img-responsive">

The FileParameterProvider lets you supply parameters in key-value files inside a Parameter Group directory.  We will discuss what a Parameter Group is
below, but for now, create a directory named "parameters" somewhere on your filesystem, and type the absolute path to this directory in the
Parameter Group Directories property.  Then, to make testing easier, select Plain Text from the Parameter Value Encoding property, and Apply.
If the specified directory is readable, you should now see a Fetch Parameters icon available.

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/04-fetch-parameters-icon.png" class="img-responsive">

### Fetching the Parameters

Click the Fetch Parameters (down arrow) icon, and a new dialog will pop up:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/05-fetch-parameters-empty.png" class="img-responsive">

We see a Parameter Group named "parameters", which is named after the directory we specified earlier.  This group can be used to create
a Parameter Context.  However, we need to create some files in order to add some parameters to this group.

Create three files inside the "parameters" directory, and give them some basic contents.  For example:

```
$ print admin > /tmp/parameters/sys.admin.username && \
  print password > /tmp/parameters/sys.admin.password && \
  print value > /tmp/parameters/sys.other
```

Each file represents a Parameter in the "parameters" group, and the contents of each file represents the parameter value.

Close the Fetch Parameters dialog, and click the Fetch Parameters button again.  Now we see some parameters!

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/06-fetch-parameters-start.png" class="img-responsive">

Notice the three filenames are now listed under Fetched Parameters in the middle column.  This just shows what parameters are available in
the group -- nothing has been applied to the flow yet.

### Creating a Parameter Context from a Parameter Group

To create our first provided Parameter Context, check the "Create Parameter Context" box.  This updates the middle column to be editable,
allowing us to select whether each parameter should be considered "sensitive" when the Parameter Context is created.  Since we only intended
the `sys.admin.password` parameter to be sensitive, we'll leave this checked, but uncheck the other two parameters.  Perhaps we also want
to name the Parameter Context something other than "parameters", so we'll update its name to "My Parameters".

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/07-fetched-parameter-sensitivities.png" class="img-responsive">

Also notice that there is now a star next to the "parameters" group.  This means the group will have an associated Parameter Context once we apply
the change.  In the future, if we have multiple parameter groups (possible in the FileParameterProvider if we specified multiple directories), we can
easily see which ones have associated Parameter Contexts by which ones are starred.

To create the Parameter Context, click Apply.

If we edit the Parameter Provider and go to the Settings tab, we'll now see a reference to the newly created Parameter Context.

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/08-referenced-parameter-context.png" class="img-responsive">

We can visit this Parameter Context in its listing by clicking on its name.  Here, notice that there is a "Go To" button in this listing, which takes us
back to the Parameter Provider.

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/09-go-to-parameter-provider.png" class="img-responsive">

However, for now we'll look at the Parameter Context by clicking the pencil button.

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/10-parameter-context-view.png" class="img-responsive">

We see the three parameters we created, along with the values of the non-sensitive ones.  Notice that we cannot edit these
parameters: only the Parameter Provider can now add, remove, or update parameters in this Parameter Context.  To see which
Parameter Provider is providing the values, go to the Settings tab:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/11-linked-parameter-provider.png" class="img-responsive">

To continue with the tutorial, click the provider name and then click the Fetch Parameters button again.

### Updating Parameter sensitivities

Now, let's say we realized we needed to make `sys.other` a sensitive parameter.  We can check the box next to this parameter, and
now we are able to click the Apply button.  Do this, and the parameter will be updated.  

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/12-updated-sensitivity.png" class="img-responsive" height="50%" width="50%">

Note that if the "My Parameters" context is assigned to a group, and that group has a component that references the `sys.other` parameter,
we will not be able to update its sensitivity.  In order to do so, we would have to first remove any references to it.

### Fetching to pull in changes from the external source

Parameters are not automatically synchronized with the external source.  To update any linked Parameter Contexts, we can
fetch the parameters again, and specify the sensitivity of any new parameters.

To simulate a few types of changes, do the following:

```
$ rm /tmp/parameters/sys.other && \
  print admin2 > /tmp/parameters/sys.admin.username && \
  print test > /tmp/parameters/new-parameter
```

This deletes the `sys.other` parameter, updates the value of the `sys.admin.username` parameter, and adds a new
parameter `new-parameter`.  Now Fetch Parameters again, and observe the changes:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/13-parameter-update.png" class="img-responsive">

We get a few "dirty" parameter indicators (the asterisk), indicating changes from the source of the Parameter Provider.  Hovering
over these asterisks gives useful information, such as "Value has changed", as we see for `sys.admin.username`.  We also see that
`new-parameter` is marked as a newly discovered parameter, indicating to us that we need to specify the sensitivity.
As before, the default sensitivity of the new parameter is "sensitive", so we'll have to adjust that if we wanted it to
be non-sensitive.  Apply the changes, and the Parameter Context will be updated to add the new parameter, remove
the `sys.other` parameter, and change the value of `sys.admin.username`.

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/14-updated-parameter-context.png" class="img-responsive">

Note that if `sys.other` was actually referenced by a component, fetching would not remove this parameter, but would
flag it as "missing".  Applying the fetched parameters would leave the parameter in place, but if at any point the
parameter was no longer referenced in the flow, the next Fetch would not return `sys.other`.  This helps to essentially
deprecate any parameters that are removed from the source without invalidating the flow when the parameters are applied.

### Composed Provided Parameter Contexts

A powerful pattern, then, is to create separate Parameter Providers for different sources, and then simply compose their created Parameter Contexts through
inheritance.  This allows sensitive parameters to originate from a secrets manager, like HashiCorp Vault, and
non-sensitive parameters to originate from other sources like Environment Variables or the filesystem.  In the following
simple example, I created an EnvironmentVariableParameterProvider, applied the parameters as non-sensitive, and then
created a FileParameterProvider and applied its parameters as sensitive.  I then created a new Parameter Context, "Parameters",
and added both of the above as inherited contexts.

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/15-composed-parameter-contexts.png" class="img-responsive">

This new Parameter Context will inherit parameters from both sources:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-providers/16-composed-parameters.png" class="img-responsive">

### Wrapping up

In conclusion, the new Parameter Providers framework is a powerful addition to NiFi's Parameter Context feature,
introducing a new extension point that can help populate flow parameters on demand.
