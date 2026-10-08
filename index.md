---
layout: home

hero:
  name: Dwesk PABX
  text: API Documentation
  tagline: Schedule outbound campaigns and surveys, upload call queues, manage DRM audio prompts, receive call webhooks as they happen, and pull your call data and IVR reports.
  actions:
    - theme: brand
      text: Quickstart
      link: /guide/quickstart
    - theme: alt
      text: API Reference
      link: /api/outbound-call

features:
  - title: Outbound Calling
    details: Schedule a CAMPAIGN or SURVEY call to any number using pre-uploaded DRM content, with DTMF branching driven by a content map.
    link: /api/outbound-call
  - title: Queue Upload
    details: Upload a CSV of lead IDs and customer numbers to schedule a dialler queue at a fixed start time.
    link: /api/queue-upload
  - title: Content Management
    details: Upload 8-bit 8 kHz mono WAV prompts to DRM and receive a contentId you can reference from any campaign.
    link: /api/content-upload
  - title: Webhooks
    details: Push events for incoming calls, pre-connect notifications and call completion with a recording URL, plus a call records feed every 5 minutes.
    link: /webhooks/
  - title: Direct Agent Mapping
    details: Assign a customer to a named agent so their calls skip the IVR and the queue and ring that agent directly.
    link: /api/direct-mapping
  - title: Recording Export
    details: Request a ZIP of call recordings for a date range and recording type, and get back a download URL.
    link: /api/recordings
  - title: Your Own Dashboard
    details: Show the Dwesk PBX dashboard numbers in your own system. Get every call once, then the changes every 5 minutes, and count them with our rules.
    link: /guide/build-your-dashboard
  - title: IVR Call Summary
    details: List the calls through an IVR flow by type and date range, the same data as the portal's IVR Call Summary, with a recording link for each answered call.
    link: /api/ivr-call-summary
  - title: IVR Reports
    details: Get IVR flow reports, missed calls and agent call reports as a CSV file straight away, with the same filters as the portal.
    link: /api/ivr-reports
---
