---
layout: article
title: Anchor's Lament
mode: immersive
header:
    theme: dark
article_header:
    type: overlay
    theme: dark
    background_color: '#79a'
    background_image:
        gradient: 'linear-gradient(135deg, #a974, #110a)'
        src: /assets/images/anchorslament/preview_banner.gif
permalink: /anchorslament/
---
<link rel="stylesheet" href="/assets/css/index.css">

<p class="gameinfo">
<b>Company:</b> Imperial Playgrounds
<br>
<b>Status:</b> In development (unreleased)
<br>
<b>Engine:</b> Unity
<br>
<b>Team size:</b> 1 programmer (me), 1 designer, various artists
<br>
<b>My participation:</b> 2 months, from project start (June - August 2025)
<br>
<b>My role:</b> Programming (gameplay, system structure, tooling, database & network)
</p>

Anchor's Lament is an unreleased Mobile/PC game by Imperial Playgrounds where I worked as the **main programmer** for 2 months during the project start in summer 2025. My job was to **get as much of the game framework up and running as possible** before I had to get back to Yrgo in August, at which point new programmers would be assigned to continue from where I left off.

The team had multiple artists and a game designer/director that implemented content into the game using my frameworks while I worked.

# My work

### Overview
- Implementation of **asynchronous combat** against other players
  - This included working with database and network code (specifically using Supabase), auth/account creation, requesting and uploading data, basic security rules and more.
- Full, modular **gameplay loop and combat system**
- Implementation of content and features in close discussion with game designer
- Editor tooling for future developers

I was responsible for the entire process from empty Unity template to working game, with intent on the development being eventually continued by other programmers after I've left.

<br>

## Modular fish action system
