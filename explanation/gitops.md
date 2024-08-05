# Gitops and Kafka

Like [Operators](./operators.md) we are not going to go in depth in the concept of gitops, just an overview and how we would apply it to Kafka and Strimzi.

## The Flow

Gitops is just the concept of using a git repository as the entry point for all operational changes. This means you have events based on the changes you make to the repo. Perhaps this is an event noting that someone created a pull request from a feature branch to main. Sometimes that means after a merge some actions need to be taken, like re-applying a bunch of yaml to a kubernetes cluster.

It also applies to things like [ArgoCD](https://github.com/catalystcommunity/documentation/blob/main/explanation/mvkube.md#how-it-is-organized) which can watch repositories and act on changes in a pull-based manner. This can be very affective at decoupling infrastructure and platforms and services from git pipelines.

Thus, when you wish to add a topic or change a user ACL or expand brokers or any other such thing, you do so with git.

## Our Requirements

Unless you have very specific reason to override these requirements, this is where we recommend starting your gitops and strimzi journey.

- Use Trunk Based development and only deploy from `main` branch, don't do a branch per environment or anything like that. If you must separate environment configs, as is often the case, do so with directories instead.
- Make base values with overrides. Many tools such as Helm, Kustomize, templating languages, `yq`, etc are available to help with this. If you need to specify 300 values for configuration, and the difference between environments is 4 values, have 300 of those in a base templated and override the 4 needed in whatever environment. Notice that you should specify a default for all 300 if you can, not 296.
- Protect your trunk. Don't let anyone, not even administrators if possible, commit to main without a pull request. Make sure your pull requests require approvals from those that own the repo/code. This is an important one. Every change can now be audited at any time, discussion on changes can be captures, and we can go back and see the results of the tests or whatnot of that PR. It takes self-discipline, but everyone involved is served by the results. Not quick enough for you? Make your pipelines faster, or have break-glass events where you can override administrator protections but a Post-Mortem is required to document why it was done and what must be done to prevent it from ever happening again.
- No middlemen. What we mean by this is that if you use gitops, the goal is autonomy through automated tooling. A service owning team should be able to make changes and deploy to every environment without involving another team. If I have to update another service, that's fine, that involves another team, but my own service I can do everything on my own. Convoluted pipelines that will make changes from one repo in another team's repo is just asking for cross-team communication problems and now you've coupled systems that shouldn't care about each other. Gitops requires automation, and automation requires decoupling.
- Push and Pull are both fine. ArgoCD can do pull based updates in Kubernetes. This is great, but not the only way and it has trade-offs. Engineering is about knowing costs and benefits and working within the constraints. If it's easier to do push based rather than ArgoCD, that's fine, but document that decision and work the flow consistently. If you use both in the same repo, too many things are involved and toes will be stepped on.

## Configuration

In cloud native vernacular, gitops follows the lessons of breaking up concerns. I have a version of code. I have a configuration. I can run the same version of code N times with their own separate configurations, or I can run the same configuration with many different versions of code, or any mix of the two sets.

This is an important concept with Strimzi and Kafka. The version of Kafka you run is not your system. The configuration is not your system. It is only a combination.

Configuration should be handled separately. Sane defaults should be provided in any code written, but the idea is those defaults are probably not production configuration, more like when we run it on our laptop.

Separate your code asset generation from your configuration application. As an example, create a docker container and put it in a registry. In a separate repo, preferably, update a helm chart that references the new container version as the default, and update the chart version itself, and then put that in a helm registry. Now you have two assets, a docker container and a helm chart. They could share the same version number, it doesn't matter, as long as they're separate assets.

Then have another git repo that defines actual implementations of configuration. Dozens of ways of doing this, including the App of Apps pattern with ArgoCD, but this separation allows some important flexibility. For instance, if you made an update in your docker container and it broke things, you could change the version number in the helm chart to the old docker container version from before the break, bump the version number of the helm chart, and now you have a rollback mechanism that's flexible. You can lean more into tools like ArgoCD to get more sophisticate rollback mechanisms in play, but the flexibility of separating assets is always required.

Make all configuration except for secret items be in git, and be the golden source of truth of what's desired. What is in the kubernetes cluster is always in flux and many hands are touching them, hopefully in tandem but not overlapping. Gitops makes git the core of what should be the state of things.

Actual environments that aren't local developer or lab environments are probably going to be larger in resources, but they should be updated frequently. Mature your processes to be able to push changes in code through to live environments as soon as it can be automatically tested.
