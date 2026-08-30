# StockGen

**A fully local pipeline that turns a list of themes into compliance-checked, submission-ready stock photography for Adobe Stock.**

StockGen runs end to end on a single Linux workstation. Using a list of themes and local AI, the software writes its own prompts, generates images, screens them for the things that get stock submissions refused, upscales survivors of human review, writes their titles and keywords, and hands over a CSV ready for Adobe Stock. Two human review gates sit in the middle to help reduce AI slop. I leaned heavily on Anthropic's Claude for software engineering, but also had a hand in shaping direction and decision-making. By default, I've put into my Claude's system prompt the following to steer it towards making secure systems:

- Always, always, ALWAYS provide code that uses secure coding techniques based on NIST (National Institute of Standards and Technology, https://www.nist.gov) and OWASP (https://owasp.org) guidance and best practices. Do not give or implement code without using and understanding these resources.
- Always gear code towards privacy. All code and even thinking should be privacy-first and privacy-conscious by design and rule.

No image or CSV leaves the machine except the final upload.

---

## Constraints that shaped it

**Fully local.** I wanted to host this pipeline locally to be in full control of data that was produced. I wanted to fully own the output and not have to worry about how the pipeline could be used if it was on someone else's server. That said, for creation of all the code for this pipeline as well as ideas and everything in between, I used Anthropic's Claude as well as my own experience and 

**Explicit Material Filter.** Since the project runs primarily using local AI on my hardware, there is a significant risk in my eyes that it might not have explicit guardrails governing adult or mature content. I know that to some extent with system prompts and the like, there may be *some* guardrails in place. However, knowing that prompt injections are a concern with AI, and frankly due to being a guy with a family, I needed to be confident that my kids would be OK seeing the output. I wanted to make sure that the images were by default, clean. As such, this is my most important requirement.

**Human review is mandatory (Absolute security necessity).** Since we are talking images in this case, I wanted at least one (but settled on two) human in the loop reviews. I wanted to reject the images that didn't pass my quality standards and I wanted to give me the best chance of my images being accepted by Adobe. I settled on two because I needed my first review to be my initial gut reaction to the images that AI produced. I wanted to focus on keeping the good images and reducing the slop (it's local AI we are talking about). I wanted my second review to be an extension of the first review, but also give me the opportunity to determine and edit the necessary information (keywords, title, etc.) for bulk uploading a .csv to Adobe.  Above all, it is just good to look at the output and not fully trust AI to provide quality outputs.

**Compliance.** I wanted to allow the software to handle most of the Adobe compliance issues like no logos or known people again giving me the best chance at my images being accepted by Adobe. My reasoning for having the software handle this as best it could followed by a human review was mainly due to that it is software...it follows rules. If I could make it do the same thing every time, it lessens the burden of me having to throw away images because they missed the mark.

---

## Architecture

```
  config/themes.txt          31 (and counting) weighted themes
         │
         ▼
  ┌──────────────────────────────────────────────┐
  │  FRONT END          (Ministral 3 · Apache-2) │
  │                                              │
  │  concept-expander    theme  → 10 concepts    │
  │  style assignment    +lighting/angle/palette │
  │  magicprompt-stock   concept → full prompt   │
  │  compliance linter   deterministic screen    │
  └──────────────────┬───────────────────────────┘
                     │  00_prompts/
                     ▼
  ┌──────────────────────────────────────────────┐
  │  GENERATION      (Z-Image Turbo · SwarmUI)   │
  │  2016 × 1152 · 16:9 · seed logged            │
  └──────────────────┬───────────────────────────┘
                     │  01_generated/
                     ▼
  ┌──────────────────────────────────────────────┐
  │  PRE-PASS                                    │
  │  corrupt check · perceptual-hash dedup       │
  │  EasyOCR text flag · sharpness flag          │
  └──────────────────┬───────────────────────────┘
                     │  02_review1_queue/
                     ▼
         ╔═════════════════════════╗
         ║  GATE 1 — human review  ║ 
         ╚═══════════╤═════════════╝
                     │  03_winners/
                     ▼
  ┌──────────────────────────────────────────────┐
  │  UPSCALE         (Real-ESRGAN · BSD-3)       │
  │  4× then downsample → 4032 × 2304 sRGB JPEG  │
  └──────────────────┬───────────────────────────┘
                     │  04_upscaled/
                     ▼
  ┌──────────────────────────────────────────────┐
  │  METADATA           (Gemma 4 12B · Apache-2) │
  │  title · keywords · category · re-linted     │
  └──────────────────┬───────────────────────────┘
                     │  05_review2_queue/
                     ▼
         ╔═══════════════════════╗
         ║  GATE 2 — metadata    ║   edit & approve
         ╚═══════════╤═══════════╝
                     │  06_outbox/  +  adobe_upload.csv
                     ▼
              manual upload → Adobe
                     │
                     ▼
              07_uploaded/<date>/
```

---

## The review interface

A single Flask application with three tabs, served by gunicorn in a rootless Podman container. To sign in, you need a token generated by the application when it runs. After browsing to this web app, you present the token to sign in. I did it this way so that I could have a way of providing some form of authentication since I can access the review area from anywhere on my network, but I also didn't want to have to deal with managing user accounts. Since this is all on my homelab, I felt the risk was relatively low in an implementation like this.

**Gate 1 Tab** is meant to be a fast visual check (the image gut reaction metioned earlier): one image on a neutral grey background with a checklist (so I can remember what I need to look at) and the ability to move winners onto gate 2. One keystroke here keeps and image while another can cut an image. Flagged images arrive ready to reject if necessary. I also had Claude create a click to zoom to 100% feature so that I can check detail.

**Gate 2 Tab** is  meant to be a careful reading of an image. The AI generates a title (that I can edit) with a character limit per Adobe's specs. Keywords are suggested and editable. Checklist items are crossed off (but can be reenabled) if AI already address the item. Titles can be regenerated on demand, with the current one passed back to the model as one to avoid. Approval is blocked while a compliance flag stands. Gate 2 has the ability to move images to the final outbox.

**Deliver Tab** This is meant to be the final stop in the pipeline so as to have a clean interface to write the bulk upload Adobe CSV of all images in the outbox and give the ability to archive/clean up my pipeline so that I can start over with a clean slate and look back on all the batches that I've uploaded over time.

---

## Pacing

A single scheduled run at 04:00 tops each queue up to a target rather than generating a fixed amount. Clear a lot and it makes a lot; clear nothing and it stands down. This pacing allows me to approve images on my schedule and not have to worry about a workload that keeps piling up (gate 2 is a real killer).

---

## Closing

This was a fun project and it allows me to produce AI generated stock imagery at scale and in an way that I can control the schedule and output. If this was going to be a production system, I would want to add user management and MFA. As such with any homelab (and access to Claude), I will likely make small improvements here and there, but for now, this pipeline is suiting my needs and I am having fun using it.
