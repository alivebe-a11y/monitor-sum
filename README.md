# monitor-sum

TrueNAS local AI health dashboard. See `SPEC.md` for the full build spec.

Stack: Glances + Scrutiny + nginx-served single-page UI that calls a local Ollama for plain-English summaries. Drop `docker-compose.yml` and the `html/` directory into a Dockge stack folder and bring it up.
