---
layout: post
title:  "Configuring Nextcloud and Collabora Code in Kubernetes"
date:   2023-02-11 11:53:00 -0500
categories: kubernetes nextcloud collabora
published: true
---
This is the third part in my series of posts regarding [running Nextcloud and Collabora CODE in Kubernetes][root-post].

1. [Deploying Nextcloud in Kubernetes][nextcloud-post]
2. [Deploying Collabora CODE in Kubernetes][collabora-post]
3. [Configuring Nextcloud with Collabora Code in Kubernetes][configuration-post]

---

## Initial Configuration

When you login to Nextcloud for the first time, you will be presented with the standard first-time-setup screens.  You can proceed with those like any other Nextcloud deployment.

Next, you will want to navigate to the top-right icon (your initials), and click "Administration Settings".  Before proceeding, fix any reported issues that may show up in "Security & setup warnings".  You may not have any warnings; a green checkbox means that you can proceed.

## Nextcloud Office Configuration

Follow these steps to configure Nextcloud Office.

1. Navigate to the top-right icon (your initials) and click "Apps".
2. In the left menu, click the "Office & text" category.  
3. Find "Nextcloud Office" and click the "Download and enable" button.
4. Navigate to the top-right icon (your initials) and click "Administration settings".
5. Click "Nextcloud Office" (sometimes, it just appears as "Office") in the left menu.
6. Click the "Use your own server" radio button.
7. Enter the domain name for your Collabora CODE server.
8. Click "Save".

If a notification comes back that "Collabora Online is reachable", then you have successfully completed the setup.  You may also update any of the other settings to meet your needs.  For instance, I have set the allow list for WOPI requests to my local network.  This forces users that will use Collabora CODE to be connected to my VPN in order for it to work -- just one additional layer of security.

Happy content creating and collaborating.

[root-post]: {% post_url 2023-02-11-nextcloud-and-collabora-on-kubernetes %}
[nextcloud-post]: {% post_url 2023-02-11-deploying-nextcloud-in-kubernetes %}
[collabora-post]: {% post_url 2023-02-11-deploying-collabora-code-in-kubernetes %}
[configuration-post]: {% post_url 2023-02-11-configuring-nextcloud-and-collabora-on-kubernetes %}
