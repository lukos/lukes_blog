---
layout: post
title: Using Talos Kubernetes cluster with Azure Entra OpenID Connect
date: '2026-06-08T16:00:00.000+00:00'
author: Luke Briner
---

## Managing Kubernetes Access
It is normal that your Developers want access to Development workloads, even if they cannot be given production access for the security reasons. On Windows Server/IIS we can just use Active Directory users and RDP but Kubernetes is a slightly different beast. What we want is:

1. An easy way to allow Developers to access a standard set of permissions on Kubernetes
1. Not having a separate auth system to maintain
1. Easy to have separate roles, perhaps "normal developer" and "lead developer" where the lead is allowed more access

Since we already use Azure Entra for authentication in Azure, this would be ideal. We create a group(s) for the relevant users, map them to cluster role(s) in Kubernetes and let them access the cluster using either Lens or Kubectl.

## The Azure Conundrum
The Azure gui is terribly complex to understand. Add an app registration and you have all kinds of stuff related to web, mobile, different flows, different scopes, redirect uris etc.

Also, the docs are massively convoluted and complicated. It is rare to find an MS document that is clear and self-contained: almost all of them link off to 50 variations of everything.

Claude was trying to be helpful but when it didn't work, I looked at the kubelogin docs, which were also a little lacking in fault-finding details.

Fortunately, I eventually worked it out with various trips back to Claude AI!

## Setup the app registration in Entra
1. Create a new App Registration
1. In Authentication, you need to make sure you have a "Mobile and desktop applications" redirect uri. You can delete the web one. Use a redirect uri like http://localhost:8000, this is what the handshake returns to and is run by kubelogin so any port that is free is fine.
1. Under Settings for the Authentication menu, tick "Allow public client flows"
1. Under "Token configuration" click "Add groups claim" and choose "Security groups" which allows you to setup roles based on groups rather than individual users.
1. Under "API permissions", add Microsoft Graph scopes for email, openid, profile and User.Read. You have to click "Add a permission" and then choose "Microsoft Graph" which should be right at the top, then search for what you want.
1. Under "Expose an API", add a scope, accepting the default name which is api://<client id> and allow Admins and Users to consent. You have to fill in some stuff to show on the consent dialog too.
1. Lastly, go to the Manifest tab and make sure that "requestedAccessTokenVersion" is set to 2, it usually defaults to null, which is old version.

## Configure Talos
You can do this on an existing cluster or modify the control plane yaml files up-front. I added a patch file like this
```yaml
cluster:
  apiServer:
    extraArgs:
      oidc-issuer-url: "https://login.microsoftonline.com/<your tenant id>/v2.0"
      oidc-client-id: "<your client id>"
      oidc-username-claim: "preferred_username"
      oidc-groups-claim: "groups"
      oidc-username-prefix: "oidc:"
      oidc-groups-prefix: "oidc:"
      oidc-signing-algs: "RS256"
```
The prefix fields are optional but allow you to distinguish between e.g. group ids that come from this and group ids that might come from other authorisation systems.

To patch your control plane, you can run e.g. `talosctl patch machineconfig -n 10.21.7.11 --patch @cp-oidc-patch.yaml` but note that this restarts the API server so don't do all control plane nodes at the same time otherwise anything connecting will fail.

## Configure Kubectl
Inside your kube config, you need something like this for the user (assuming you already set up the cluster certificate data etc)
```yaml
- name: developer@cluster-dev
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: C:\Users\luke\.azure-kubelogin\kubelogin
      args:
        - get-token
        - --login
        - devicecode
        - --tenant-id
        - <tenant id>
        - --client-id
        - <client id>
        - --server-id
        - <client id too!>
        - --redirect-url
        - https://localhost:8001
```
> The redirect uri needs to match what you added in Azure and also be careful about tabs vs spaces since some text editors might automatically add tabs for you (kubectl will usually complain). Also note the full path to kkubelogin so that Lens can find it.

When you first connect to your cluster with kubectl, it should show you a url to put into your browser and authenticate against. This is sometimesa little easier that using the full browser interactive mode which you can get with "--login interactive" instead of "--login device code"

## Linking users to roles in Talos
It is usually best to use Entra groups for this and there are two main types of roles. "Cluster roles" apply across namespaces and are more aimed at people who manage the cluster. Roles on the other hand are namespace scoped so provide better fine-grained access for Developers only into the namespaces they should be able to access.

These are mapped to groups (or users) with "Cluster Role Bindings" or "Role Bindings" both of which have a similar syntax e.g.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ss-aks-developers
subjects:
  - kind: Group
    apiGroup: rbac.authorization.k8s.io
    name: oidc:12345678-abcd-efgh-ijkl-123456789012
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ss-namespaces-readonly
```
Where the subject "name" is the group id from Azure with the prefix we chose earlier. The RoleBinding is the same except it has a namespace under metadata and is type "RoleBinding"

You will notice that you can map multiple subjects to the same role or you can just create separate ClusterRoleBindings, whatever you find easier to manage.

Once deployed, all role bindings are merged to define what roles/permissions a user is given. This usually happens automatically after about 30 seconds without having to re-authenticate or anything.

## Resources and verbs
The syntax for roles and cluster roles is simple enough except the exact names are not always obvious so sometimes you just wait for an error to tell you what the actual name of a resource is!

This is an example of a role:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: view-role
  namespace: development
rules:
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
    resources:
      - pods
      - pods/log
      - pods/exec
      - pods/attach
      - services
      - configmaps
      - ingresses
      - deployments
```
It might not be obvious that you need "watch" to see the logs of a pod but if you try and view them, it will tell you that you don't have "watch" permission. Also, you need a blank string for apiGroups to say that you are allowed access to all apiGroups (most small clusters only have 1 anyway).

I expect your favourite LLM can help you with any issues you are having!

## Using with Lens
Initially it seemed completely broken in Lens until I did two things (either of which might have fixed it).

1. Make sure the full path to kubelogin is in your kube config since Lens doesn't pick up your own path.
1. After closing Lens, go into Task Manager and kill off the background processes it is running!
1. Restart Lens