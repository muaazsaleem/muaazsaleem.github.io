---
layout: post  
title: "The 2 main parts of a Kubernetes Validating Admission Webhook"  
date: 2020-11-06
---

## Validating Admission Webhooks

Validating Admission Webhooks can be used to intercept requests to the API Server and validate new and updating resources. For example, you could have a Validating Admission Webhook that keeps new `Deployment` objects from being created if they're missing an `application` label. You could also keep certain resources from being updated.

* A Validating Admission Webhook has 2 main parts:
    * A Webhook Server
    * A `ValidatingAdmissionWebhookConfiguration`[resource][0]


> Note: The code examples and a couple passages in this post are borrowed from the [Dynamic Admission Control][1]  reference. Shared under the [Creative Commons Attribution 4.0 International][2]  Lisence.

## The Webhook Server

The webhook server is just a webserver that handles "validation" requests from the API server.

[0]: https://v1-18.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#validatingwebhookconfiguration-v1-admissionregistration-k8s-io
[1]: https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/
[2]: https://creativecommons.org/licenses/by/4.0/

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/7xli7795fe2ayecwlgh8.png)


These requests are regarding specific resource types and object changes. For example, creation of a Deployment resource. This is specified in the `ValidatingAdmissionWebhook` for the admission webhook.

The webserver communicates with the API Server over HTTPs (`TLS`) and therefore requires a TLS cert and key pair.

### Requests

Webhook Servers a.k.a webhooks, are sent a `POST` request, with `Content-Type: application/json`, with an `AdmissionReview.Request` [API object][1] in the `admission.k8s.io` API group serialized to `JSON` as the body.

This example shows the _abridged_ data contained in an `AdmissionReview` object for a request to update the scale subresource of an `apps/v1 Deployment`:

    {
      "apiVersion": "admission.k8s.io/v1",
      "kind": "AdmissionReview",
      "request": {
        # Random uid uniquely identifying this admission call
        "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
        # Name of the resource being modified
        "name": "my-deployment",
        # Namespace of the resource being modified, if the resource is namespaced (or is a Namespace object)
        "namespace": "my-namespace",
        # operation can be CREATE, UPDATE, DELETE, or CONNECT
        "operation": "UPDATE",
        # object is the new object being admitted.
        # It is null for DELETE operations.
        "object": {"apiVersion":"autoscaling/v1","kind":"Scale",...},
        # oldObject is the existing object.
        # It is null for CREATE and CONNECT operations.
        "oldObject": {"apiVersion":"autoscaling/v1","kind":"Scale",...},
        # dryRun indicates the API request is running in dry run mode and will not be persisted.
        # Webhooks with side effects should avoid actuating those side effects when dryRun is true.
        # See http://k8s.io/docs/reference/using-api/api-concepts/#make-a-dry-run-request for more details.
      }
    }

### Response

The webhook server responds with a `200` HTTP status code, `Content-Type: application/json`, and a `body` containing an `AdmissionReview.Response` object (in the same version they were sent), with the response stanza populated, serialized to `JSON`. At a minimum, the response stanza must contain the following fields:

* `uid`, copied from the `request.uid` sent to the webhook
* allowed, either set to `true` or `false`


Example of a minimal response from a webhook to allow a request:

    {
      "apiVersion": "admission.k8s.io/v1",
      "kind": "AdmissionReview",
      "response": {
        "uid": "<value from request.uid>",
        "allowed": true
      }
    }

Example of a response to forbid a request, customizing the HTTP status code and message:

    {
      "apiVersion": "admission.k8s.io/v1",
      "kind": "AdmissionReview",
      "response": {
        "uid": "<value from request.uid>",
        "allowed": false,
        "status": {
          "code": 403,
          "message": "You cannot do this because it is Tuesday and your name starts with A"
        }
      }
    }

## ValidatingWebhookConfiguration

The `ValidatingAdmissionWebhookConfiguration` specifies what object type and request methods should the webserver be called on. The `Configuration` resource also specifies the TLS `caBundle` the API server should use to connect with the webserver.

An example `ValidatingWebhookConfiguration`:

    apiVersion: admissionregistration.k8s.io/v1
    kind: ValidatingWebhookConfiguration
    metadata:
      name: "pod-policy.example.com"
    webhooks:
    - name: "pod-policy.example.com"
      rules:
      - apiGroups:   [""]
        apiVersions: ["v1"]
        operations:  ["CREATE"]
        resources:   ["pods"]
        scope:       "Namespaced"
      clientConfig:
        service:
          namespace: "example-namespace"
          name: "example-service"
        caBundle: "Ci0tLS0tQk...<`caBundle` is a PEM encoded CA bundle which will be used to validate the webhook's server certificate.>...tLS0K"
      admissionReviewVersions: ["v1", "v1beta1"]
      sideEffects: None
      timeoutSeconds: 5

## Further Resources

* To learn more about validating admission webhooks and dynamic admission control in general, see the [Dynamic Admission Control][2] reference from the [Kubernetes docs][3].
* You can find the definitions of the `AdmissionReview` object in [k8s.io/api][4] repository.
* I also like to look at the Kubernetes API reference. You can find the `ValidatingAdmissionConfiguration` spec [there][5].


- - -

[0]: ./1603316619167.png
[1]: https://github.com/kubernetes/api/blob/c59710ee51e70325197298cd6243d32f1aa85df0/admission/v1/types.go#L33
[2]: https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/
[3]: https://kubernetes.io/docs/home/
[4]: https://github.com/kubernetes/api/blob/c59710ee51e70325197298cd6243d32f1aa85df0/admission/v1/types.go#L40
[5]: https://v1-18.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#validatingwebhookconfiguration-v1-admissionregistration-k8s-io