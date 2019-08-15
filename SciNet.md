# Running Slurm Jobs on SciNet Niagara #
## Getting Set Up ##

Now that you know a bit about how the Kimel Lab Slurm cluster works (if not see [here](https://github.com/TIGRLab/TIGRSlurm-Docs/blob/master/README.md)), we can discuss some of the differences between the local lab cluster and the SciNet Niagara cluster.

All of our publically available datasets are processed, archived, and stored on SciNet. You will do all of your computing on SciNet for all of the following studies:

- HBN (Healthy Brain Network dataset)
- HCP (Human Connectome Project dataset)
- PNC (*The something something something dataset Jerry what is this?*)
- POND (Province of Ontario NeuroDevelopmental Disorders dataset)

To start with, we'll assume you have a Compute Canada account associated with the lab's RAC. If not, or if you don't know, visit [FAKELINKHERE](SciNet.onboarding.docs "This will be a link to SciNet onboarding documentation."). To access SciNet, can `ssh` into it using your Compute Canada account username and password:

```
$ ssh <your_CC_username>@niagara.scinet.utoronto.ca

> <your_CC_username>@niagara.scinet.utoronto.ca's password:
```

Once you type your password, you will be on a login node. There are seven such and you'll get routed to one of them, and it doesn't really matter which one.
