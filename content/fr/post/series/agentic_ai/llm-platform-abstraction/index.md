+++
author = "Smaine Kahlouch"
title = "Self-hosted LLM stack : quand l'abstraction grandit"
date = "2026-07-12"
summary = "Deux mois après avoir posé les fondations, l'abstraction `InferenceService` a révélé quatre manques : deux sources de vérité, aucune stratégie de rollout, aucune échappatoire, aucune découvrabilité. Voici comment on les a comblés — canary LoRA à coût GPU nul compris."
featured = true
codeMaxLines = 30
usePageBundles = true
toc = true
draft = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "vllm", "crossplane", "platform-engineering"]
thumbnail = "thumbnail.png"
+++

{{% notice info "Série Agentic AI — Partie 4" %}}
Cet article fait suite à [Self-hosted LLM stack : poser les fondations](/fr/post/series/agentic_ai/llm-self-hosted-stack/) (partie 3), qui décrit la plateforme sur laquelle tout ce qui suit s'appuie. **Ici, on la fait vivre** — et on répare ce qui a cassé.
{{% /notice %}}

## :dart: Objectifs

## :mag: Ce qui clochait

## :door: Le routage rejoint le modèle

## :dna: LoRA en deux minutes

## :hatching_chick: Canary : 10 % du trafic, zéro GPU

## :unlock: L'échappatoire `engineArgs`

## :eyes: Le modèle dit ce qu'il sert

## :bar_chart: Mesurer le canary

## :compass: Endpoint Picker : au-delà du round-robin

## :telescope: Modelplane : évolution convergente

## :thought_balloon: Dernières remarques

## :bookmark: Références
