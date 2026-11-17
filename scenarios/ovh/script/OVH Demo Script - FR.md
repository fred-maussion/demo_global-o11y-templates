# OVHcloud Network Operations — Script de demo complet

Elastic Observability + Reponse aux incidents assistee par IA sur infrastructure OVHcloud

Incident principal (HITL)

CH-01 : vRack VLAN Segmentation

Incident secondaire (Auto)

CH-12 : LB Health Check Flap

Code erreur

VRACK-VLAN-SEGMENTATION

Duree

\~12 minutes

Avant de commencer : injecter **CH-01 (vRack VLAN Segmentation)** en mode Cascading Failure depuis la Chaos Console. Injecter egalement **CH-12 (LB Health Check Flap)** en mode Calibration Drift — celui-ci s'auto-remedie en arriere-plan et sera revele a la fin.

---

## Acte 1 — L'impact metier

Application Cockpit

Dans l'environnement actuel, chaque application a sa propre architecture — des monolithes historiques aux microservices cloud-native, deployes on-premise, dans le cloud ou dans des environnements hybrides. Les observer toutes a travers un seul tableau de bord est deja un defi ; mais correler les signaux techniques avec l'impact business tout en evaluant la consommation du budget d'incidents est critique pour toute entreprise. \
\
La capacite unique d'Elastic a ingerer tout type de donnee dans sa base Search AI en fait le socle ideal pour une vue a 360 degres. \
\
En regardant notre cockpit applicatif, qui combine signaux techniques et business pour montrer l'etat de bout en bout de nos operations reseau OVHcloud, on constate que notre SLO applicatif brule rapidement et impacte directement le business. La latence cross-DC du vRack a explose, la sante des backends chute, et notre SLO d'uptime est en breach. \
\
En tant que SRE, notre mission est d'analyser l'incident, d'executer la remediation et de livrer une analyse de cause racine — le tout pendant que les dirigeants suivent le SLO en temps reel.

Montrer les tuiles KPI : la **latence cross-DC du vRack** est passee de \~5ms a 100ms+, le **Backend health %** est tombe sous 60%, l'**Uptime %** est en breach SLO. Les graphiques de tendance montrent le moment exact de l'injection de la panne.

---

## Acte 2 — Trouver l'aiguille dans la botte de foin

Discover

Naviguer vers **Discover**. Dans l'editeur ES|QL, saisir la requete du **Tab 1** ci-dessous.

Tab 1 — Source de donnees

Tab 2 — Categorisation ML

```
FROM logs.otel.ovhnet, logs.otel.ovhnet.*, logs.ecs.ovhnet, logs.ecs.ovhnet.*
// Focus on ERROR log messages
```

```
FROM logs.otel.ovhnet, logs.otel.ovhnet.*, logs.ecs.ovhnet, logs.ecs.ovhnet.*
| WHERE log.level == "ERROR"
| STATS count=COUNT() BY category=CATEGORIZE(body.text)
| SORT count DESC
```

Dans l'environnement actuel, le parcours d'un SRE peut commencer n'importe ou dans Elastic. Voyons comment Elastic permet aux SRE de trouver l'aiguille dans la botte de foin parmi le volume massif de logs recus chaque jour, grace aux capacites agentiques d'Elastic. \
\
Elastic offre une exploration approfondie des donnees a travers differents signaux via Kibana et ES|QL, notre langage de requete. S'il y a toujours une courbe d'apprentissage avec toute nouvelle solution, les capacites agentiques d'Elastic abaissent cette barriere en traduisant le langage naturel en requetes — rendant l'exploration des donnees accessible a tous. \
\
Voyons cela en action. Avec la requete ES|QL du **Tab 1**, je cible uniquement le namespace reseau OVH — filtrant ainsi tout autre scenario en cours sur la plateforme. On voit immediatement un pic soudain dans le volume de logs.

Focus on ERROR log messages

Grace aux capacites IA d'Elastic, j'exprime simplement mon intention et l'assistant la traduit dans le bon filtre ES|QL. On voit une forte augmentation des messages ERROR — mais comment identifier quelle erreur specifique provoque ce pic ? \
\
C'est la que la categorisation par Machine Learning entre en jeu. En passant au **Tab 2**, j'utilise la fonction `CATEGORIZE()` d'ES|QL pour regrouper automatiquement les logs d'erreur par pattern et les trier par nombre — identifiant la categorie exacte derriere la montee en charge.

Basculer vers la requete du **Tab 2** dans l'editeur ES|QL — ou cliquer sur **"Machine Learning Categorisation"** dans Discover.

Parfait — on voit maintenant clairement quelle categorie provoque le pic. Plongeons dans l'investigation.

Cliquer sur le **bouton d'expansion** sur une trace contenant l'erreur.

Le champ `body.text` contient des donnees non structurees ou se trouve notre message d'erreur.

Developper le message pour reveler l'erreur complete.

`VRACK-VLAN-SEGMENTATION` semble etre la raison derriere le pic. Le log nous indique qu'il y a un desaccord de tagging VLAN entre les datacenters — les instances de GRA11 ne peuvent pas joindre les instances de SBG5 via le reseau prive vRack, avec 85% de perte de paquets. \
\
Mais qu'est-ce que cela signifie concretement, et d'ou ca vient ? La reponse se trouve dans les traces applicatives, ou les donnees sont correlees a travers l'ensemble de l'application — rendant la transaction lisible de bout en bout. \
\
Et si on pouvait demander a notre agent de realiser une premiere analyse d'erreur ?

---

## Acte 3 — Analyse assistee par IA

AI Assistant

Cliquer sur **"Explain this log entry"**.

Grace au Context Engine IA d'Elastic, l'agent dispose du contexte complet et de competences specialisees pour executer une analyse detaillee — determinant une cause racine initiale a partir du message d'erreur non structure, evaluant l'impact et proposant les prochaines etapes appropriees. \
\
L'agent identifie que l'erreur `VRACK-VLAN-SEGMENTATION` est causee par un desaccord de tagging VLAN entre les datacenters GRA et SBG. Il voit la cascade : la segmentation vRack rend les backends de l'OVHcloud Load Balancer a SBG injoignables, les pods MKS passent en CrashLoopBackOff car les services cross-DC sont coupes, et le Gateway NAT reroute le trafic sortant via l'internet public en mode fallback. \
\
Cela nous donne une solide analyse de premier niveau. Maintenant, transformons cela en un case officiel pour lancer un incident formel.

Cliquer sur **"Start conversation"**.

On entre maintenant dans une experience conversationnelle ou l'on peut interagir directement avec notre agent OVHcloud Network Analyst, qui conserve le contexte de l'analyse precedente pour agir sur nos demandes. Ici, on souhaite creer un dashboard pour suivre ce message d'ERROR specifique afin d'avoir des preuves claires une fois le probleme resolu. Demandons-lui de le construire.

Create a dashboard to track progress of this VRACK-VLAN-SEGMENTATION issue

Une fois cree, cliquer sur l'**apercu du dashboard**.

Grace a l'agent, nous disposons maintenant d'un dashboard dedie pour suivre la resolution de notre probleme de segmentation VLAN vRack — affichant le taux d'erreur dans le temps, les datacenters affectes et les tendances de latence cross-DC.

Sauvegarder le dashboard.

---

## Acte 4 — Creation du case et runbook

Human-in-the-Loop

On peut maintenant transformer ce contexte et ce dashboard en un case actionnable pour suivre la resolution et construire le RCA final. Demandons a l'agent de s'en charger.

Create an observability case and link the dashboard to track the resolution. Find the appropriate remediation actions from the internal runbook.

Cliquer sur **"Go to case"**.

Toutes les conclusions ont ete collectees et consolidees dans un seul case complet — contenant non seulement les details de notre investigation, mais aussi les etapes de remediation exactes extraites de notre runbook interne. \
\
Le runbook prescrit de verifier les VLAN IDs a travers tous les datacenters en utilisant le CLI OVH et les commandes OpenStack, de verifier la coherence de la configuration vRack, et si necessaire, de re-appliquer le tagging VLAN via l'API OVH. Le runbook indique que cette remediation necessite une approbation manuelle — le vRack est l'epine dorsale de notre reseau prive, toute reconfiguration doit etre validee par un SRE.

---

## Acte 5 — La revelation : l'auto-remediation tournait deja

Auto-Remediate

Jusqu'a present, nous avons vu comment les capacites IA d'Elastic permettent aux SRE d'accelerer drastiquement le temps moyen de detection et de resolution. Mais et si le systeme pouvait se guerir tout seul ? \
\
C'est en fait ce qui se passait en arriere-plan pendant notre investigation : un second incident — `LB-HEALTHCHECK-FLAP` — a ete detecte, trie et analyse automatiquement, avec une analyse de cause racine complete et un score de confiance etaye par des preuves. Les health checks HAProxy oscillaient a cause de sondes de readiness des backends echouant durant la perturbation vRack. Le systeme a identifie cela comme une condition auto-guerissante : une fois l'oscillation stabilisee, les backends se retablissent automatiquement via le mecanisme rise/fall natif de HAProxy. \
\
Tout cela a ete conditionne dans un case, attendant simplement que le SRE approuve la remediation du runbook pour le probleme principal du vRack. \
\
Trop beau pour etre vrai ? Allons voir.

Cliquer sur la **notification email** dans votre boite de reception → cliquer sur le **Case** (montrer brievement le contenu) → cliquer sur **"conversation"** → **valider la remediation**.

Le workflow a maintenant execute le plan de remediation base sur le runbook interne et suivi la progression — gardant le case continuellement a jour jusqu'a la resolution complete. Conformement aux directives du runbook, cette remediation a impose une validation humaine avec un controle "human-in-the-loop" avant de declencher le correctif de reconfiguration vRack.

Revenir au **case ou au cockpit** et observer la resolution. La latence cross-DC du vRack revient a la normale, la sante des backends remonte a 100%, le SLO d'uptime arrete de bruler.

---

## Acte 6 — Conclusion

Dans cette demonstration, vous avez vu comment les capacites agentiques natives d'Elastic permettent aux SRE d'aller de la detection de l'aiguille dans les logs jusqu'a la resolution complete d'incident sur une infrastructure OVHcloud — tout en conservant le controle humain sur les remediations critiques. \
\
Du reseau prive vRack au load balancing HAProxy, du scrubbing Anti-DDoS VAC a la connectivite hybride OVHcloud Connect — Elastic observe tout a travers une seule plateforme, correle les signaux entre services et datacenters, et utilise l'IA pour accelerer chaque etape du cycle de vie des incidents.

---

## Annexe — Chronologie de la demo

**T-0:00** — Injecter CH-01 (vRack VLAN Segmentation, HITL) + CH-12 (LB Health Check Flap, Auto) depuis la Chaos Console

**T+0:30** — Ouvrir l'Application Cockpit, montrer le SLO qui brule et l'impact KPI

**T+1:30** — Naviguer vers Discover, executer la requete ES|QL Tab 1 sur le namespace ovhnet, montrer le pic de logs

**T+3:00** — Executer la requete ES|QL Tab 2 (CATEGORIZE) → identifier VRACK-VLAN-SEGMENTATION

**T+4:00** — Developper l'entree de log, montrer body.text, cliquer sur "Explain this log entry"

**T+5:30** — Analyse IA : cause racine, impact en cascade, prochaines etapes

**T+6:30** — Start conversation → demander a l'agent de creer un dashboard de suivi

**T+8:00** — Demander a l'agent de creer un case avec remediation runbook

**T+9:30** — Revelation : l'auto-remediation de CH-12 tournait en arriere-plan

**T+10:30** — Notification email → approuver la remediation HITL pour CH-01

**T+11:30** — Observer la resolution sur le cockpit : les metriques se retablissent, le SLO arrete de bruler

**T+12:00** — Conclusion

---

## Annexe — Correspondance OVH / GCP pour les presentateurs

Si vous avez deja presente la version GCP, voici comment les elements correspondent :

| Element | Version GCP | Version OVH |
| --- | --- | --- |
| Incident principal | `VPC-ROUTE-LIMIT-EXCEEDED` | `VRACK-VLAN-SEGMENTATION` |
| Auto-remediation | CH-16 Backend Unhealthy | CH-12 LB Health Check Flap |
| Cause racine | Limite de 250 routes VPC peering | Desaccord de tagging VLAN vRack entre DCs |
| CLI du runbook | `gcloud compute networks peerings list-routes` | `ovh-cli vrack get` / `openstack network list` |
| Impact KPI | CDN cache hit rate, Cloud Armor blocks | Latence cross-DC vRack, Backend health % |
| Cascade | VPC → LB → NAT | vRack → LB → MKS |
| Agent IA | Google Cloud Network Analyst | OVHcloud Network Analyst |