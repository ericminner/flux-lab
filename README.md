# flux-lab

A timeboxed Flux lab on Rancher Desktop: about 2 h 40 min. Each step has a time cap,
a prediction to make before you run anything, and a done check. When a cap runs out,
write down where you got to and move on.

Rules:

- You run every command.
- Before each **Predict** item, say your answer out loud or write it in `NOTES.md`.
- Don't look at the `drill-*` branches (`git diff`, `git log -p`, GitHub's compare
  view). They hold the faults for step 5.
- Stuck for more than 10 minutes? Ask Claude for a hint, one level at a time.

On AKS, Flux usually comes from the `microsoft.flux` cluster extension
(`az k8s-configuration flux create`). Azure installs the controllers and creates the
`GitRepository` and `Kustomization` objects. This lab mirrors that: `flux install`
stands in for the extension, and you write the two objects yourself.

## 0. Cluster and CLI (10 min)

1. Rancher Desktop → Preferences → Kubernetes: enable Kubernetes. Under Virtual
   Machine, give it at least 4 GB of memory.
2. Install the CLI, point kubectl at Rancher, and check the cluster:
   ```sh
   brew install fluxcd/tap/flux
   kubectl config use-context rancher-desktop
   flux check --pre
   ```
3. Write the output of `flux version --client` and `kubectl version` in `NOTES.md`.

**Done when:** `flux check --pre` says all prerequisites pass.

## 1. Your Git repo (10 min)

1. On GitHub, create an **empty public** repo named `flux-lab`, with no README or
   license.
2. From this folder:
   ```sh
   git remote add origin https://github.com/ericminner/flux-lab.git
   git push -u origin main drill-1 drill-2 drill-3
   git checkout main
   ```

**Done when:** GitHub shows `main` and the three drill branches.

## 2. Install Flux and write the two objects (30 min)

1. Install the controllers:
   ```sh
   flux install
   kubectl get pods -n flux-system
   ```
2. Fill in the `TODO(eric)` blocks in `clusters/rancher/source.yaml` and
   `clusters/rancher/podinfo.yaml`. Use the docs links in each file.
3. **Predict:** after you apply both files, which controller acts first, what does it
   produce, and how does the other controller get it? What appears in the `podinfo`
   namespace, and in what order?
4. Apply them and watch:
   ```sh
   kubectl apply -f clusters/rancher/
   flux get sources git
   flux get kustomizations --watch
   ```
5. Look at the result:
   ```sh
   kubectl get all -n podinfo
   flux tree kustomization podinfo
   kubectl port-forward -n podinfo svc/podinfo 9898:9898   # open http://localhost:9898
   ```
6. Commit and push your two files to `main`.

**Done when:** `flux get all -A` shows Ready=True for the source and the Kustomization,
both podinfo pods are Running, and the UI shows "flux-lab: main".

## 3. Change Git and watch it sync (20 min)

1. In `apps/podinfo/deployment.yaml`, change `PODINFO_UI_COLOR` to `"#2e7d32"` and
   `PODINFO_UI_MESSAGE` to `"flux-lab: change 1"`. Commit and push to `main`.
2. **Predict:** how long until the UI changes, at the slowest? Which two intervals
   add up to that?
3. Time it with `flux get kustomizations --watch` and `kubectl get pods -n podinfo -w`.
4. Make a second change (a new message). This time don't wait:
   ```sh
   flux reconcile kustomization podinfo --with-source
   ```
   Why `--with-source`? What happens without it?
5. Compare the revision in `flux get sources git` with `git rev-parse HEAD`.

**Done when:** the UI shows your second message, and the source revision matches
your latest commit.

## 4. Edit the cluster by hand (40 min)

Do these in order on `main`. For each row, write your prediction **before** running
it, then wait a minute (or reconcile) and record what happened. Some outcomes
surprise most people, so test them rather than assume.

| # | Command | Predict | Result |
| --- | --- | --- | --- |
| a | `kubectl scale deploy/podinfo -n podinfo --replicas=5` | | |
| b | `kubectl delete deploy/podinfo -n podinfo` | | |
| c | `kubectl set image deploy/podinfo -n podinfo podinfo=ghcr.io/stefanprodan/podinfo:6.16.0` | | |
| d | `kubectl rollout history deploy/podinfo -n podinfo`, then `kubectl rollout undo deploy/podinfo -n podinfo` | | |
| e | `kubectl annotate deploy/podinfo -n podinfo kubernetes.io/change-cause="manual test"` | | |
| f | `kubectl label deploy/podinfo -n podinfo hacked=true` | | |
| g | `kubectl annotate deploy/podinfo -n podinfo kustomize.toolkit.fluxcd.io/reconcile=disabled`, then repeat (a) | | |
| h | Remove that annotation (`kubectl annotate deploy/podinfo -n podinfo kustomize.toolkit.fluxcd.io/reconcile-`) | | |
| i | `flux suspend kustomization podinfo`, repeat (a), then `flux resume kustomization podinfo` | | |
| j | `git rm apps/podinfo/service.yaml`, remove it from `apps/podinfo/kustomization.yaml`, push, reconcile; then restore both and push | | |

Useful while you test:

```sh
kubectl get deploy podinfo -n podinfo -o yaml --show-managed-fields | less   # who owns each field
kubectl get kustomization podinfo -n flux-system -o yaml                      # the reconcile annotation flux sets
flux diff kustomization podinfo --path ./apps/podinfo                         # Git vs cluster, from your laptop
```

Afterwards, read "Controlling the apply behavior of resources" and "Suspending and
resuming" in the [Kustomization docs](https://fluxcd.io/flux/components/kustomize/kustomizations/)
and check your results against them.

**Done when:** every row has a result, and you can explain (d), (f) and (g) in one
sentence each.

## 5. Drills (45 min, 15 min each)

Each drill is a ticket. Point Flux at the branch, then diagnose it **cold**,
narrating out loud the way you would in the interview.

```sh
kubectl -n flux-system patch gitrepository flux-lab --type merge \
  -p '{"spec":{"ref":{"branch":"drill-1"}}}'
flux reconcile kustomization podinfo --with-source
```

The drill branches start from the original commit, so the UI colour goes back to
blue as well. Ignore that.

| Branch | Ticket |
| --- | --- |
| `drill-1` | "We moved the UI settings into a ConfigMap and pushed. Flux is unhappy and the new message never showed up." |
| `drill-2` | "Someone tidied the deployment labels. Since then Flux shows Ready=False and nothing deploys." |
| `drill-3` | "Released the new podinfo version with a new colour. The pods look stuck." |

For each drill:

1. Start a 15-minute timer.
2. Find the root cause using only cluster and Flux tooling: `flux get all -A`,
   `flux events -n flux-system --for Kustomization/podinfo`,
   `kubectl describe kustomization podinfo -n flux-system`, then `get`, `describe`,
   `logs` and `events` on the workloads. Read the repo files last, to confirm.
3. Fix it **in Git**: check out the drill branch, commit and push. Watch Flux
   recover.
4. In `NOTES.md`, write the symptom, the root cause, the command that showed it, and
   what you'd check first next time.
5. Answer: while it was broken, what was serving traffic, and why?

When you're done with all three, point the source back at `main`:

```sh
kubectl -n flux-system patch gitrepository flux-lab --type merge \
  -p '{"spec":{"ref":{"branch":"main"}}}'
```

## 6. Say it out loud (10 min)

- "It's 3am, errors are spiking on a Flux-managed service. Walk me through it."
  Cover: what just merged, `flux get`, rollback by `git revert` versus
  `flux suspend` plus a hand fix, who you tell, and when you escalate.
- Push-based CD (CI runs `kubectl apply`) versus pull-based (Flux in the cluster):
  credentials and blast radius.
- `GitRepository` versus Flux `Kustomization` versus `kustomization.yaml` versus
  `HelmRelease`.
- How would you keep secrets in a GitOps repo? (SOPS, Sealed Secrets, or Key Vault
  with workload identity on AKS.)

## 7. Clean up (5 min)

```sh
flux uninstall
kubectl delete namespace podinfo --ignore-not-found
```

Turn Rancher's Kubernetes back off if you need the memory for Compose.
