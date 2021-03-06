# Synchronising a GitHub Repository into Bitbucket.

This project demonstrates maintaining code in Github and having it automatically update changes into a BitBucket mirror repository. This is entirely driven from the GitHub end and only requires a github action and a secret containing bitbucket access credentials. 

This is a one way synch. If you modify the BB version either the action will fail due to conflicts or the next update will replace your changes.  Hence you should configure the Bitbucket repo to be read only except for the Github account doing the updates. 

The action script syncs all branches on push.  You might want to modify the script to only synch on push to master. 

## Steps where the synch target requires SSH access.
These instructions will setup a SSH access connection between an org or repo on Github to its equivalent on BitBucket. (if you don't need SSH or are using https access see below.)

### create an SSH keypair and add to Bitbucket
Github now supports org level secrets so I suggest setting up a single BITBUCKET_SSH_KEY at the org level and then all repos owned by the org will be able to be synched. Otherwise copy the SSH private key into the secrets of each repo.

https://confluence.atlassian.com/bitbucket/set-up-an-ssh-key-728138079.html#SetupanSSHkey-ssh2

You do not want to use your home .ssh/id_rsa default key pair for this - the private key is going to be placed in a github secret, if you need to change the pair you don't want to break other keys you use.  Also you may want to copy the repo to different targets. I suggest creating a key named for the target repo owner. 

To create a key with a name or path other than the default, specify the full path to the key. For example, to create a key called my-new-ssh-key, enter a path like the one shown at the prompt:

    Warning - the ssh library in the action requires an SSH private key in PEM format that starts with -----BEGIN RSA PRIVATE KEY-----. 


````
$ ssh-keygen -P "" -t rsa -b 4096 -m pem -f pfrnz
````

This generates two files
pfrnz - containing the private key
pfrnz.pub - containg the public key

### Add the _public_ key to Bitbucket.
On Bitbucket you can add the ssh key at the workspace, project or repo level. 
e.g - goto https://bitbucket.org/<org>/workspace/settings/ssh-keys
add the _public_ key.  


#### Testing Bitbucket SSH access
On the local device move the keys to the ~/.ssh folder and try checking out a repo using the SSH URL. ( Note: you should also set the access right to the private key to 400. e.g `chmod 400 .ssh/groatnz )
e.g. 
`git clone groatnz@bitbucket.org:groatnz/bitbucket-integration.git`

Update the remote URL with your Bitbucket username by replacing git@bitbucket.org with <username>@bitbucket.org. For this step and the ones that follow, enter your username in place of <username>.

`git remote set-url origin groatnz@bitbucket.org:groatnz/bitbucket-integration.git`

### add the _private_ key to a github secret
On Github place the private key into a repo secret. 
e.g at `https://github.com/pfrtest/bitbucket-integration/settings/secrets/new`

Add a new secret `BITBUCKET_SSH_KEY` with the private key

#### create a known hosts entry for bitbucket
Set known_hosts option correctly (use ssh-keyscan command).
`ssh-keyscan -f pfrnz bitbucket.org`

capture the output which will look like 
````
bitbucket.org ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAubiN81eDcafrgMeLzaFPsw2kNvEcqTKl/VqLat/MaB33pZy0y3rJZtnqwR2qOOvbwKZYKiEO1O6VqNEBxKvJJelCq0dTXWT5pbO2gDXC6h6QDXCaHo6pOHGPUy+YBaGQRGuSusMEASYiWunYN0vCAI8QaXnWMXNMdFP3jHAJH0eDsoiGnLPBlBp4TNm6rYI74nMzgz3B9IikW4WVK+dc8KZJZWYjAuORU3jc1c/NPskD2ASinf8v3xnfXeukU0sJ5N6m5E8VLjObPEO+mN2t/FZTMZLiFqPWc/ALSqnMnnhwrNi2rbfg/rd/IpL8Le3pSBne8+seeFVBoGqzHM9yXw==
````

Add a new secret `BITBUCKET_KNOWN_HOSTS` with this value.

### test the action script sync-bitbucket-ssh.yml
Add the destination repo name to the action script. 

### Commit
Push your code to the github branch. The action should run and automatically synch the changes to the bitbucket repository. 



## Steps for an existing Github repository - HTTPS & simple credentials

1. Checkout the repo, create a .github/workflows folder and copy the sync-bitbucket.yml file there.
2. On Bitbucket create a new empty repository in the project and workspace you want to use. You should name the repo the same as on github but its not absolutely necessary. Get the URL for the new repo. e.g. `https://<username>:<password>@bitbucket.org/pfr/example-repo.git`. If the repo is public you will not require the username and password. 
3. For private repos, on Github in the Settings tab, add a secret with name BITBUCKET_SYNC_URL and value being the URL. This is used to set the sync remote for the checkout to the bitbucket repo. 
    `git remote add sync ${{ secrets.BITBUCKET_SYNC_URL }}` 
4. For public repos - you can do the same or use a literal string in the action.
    `git remote add sync https://bitbucket.org/pfr/example-repo.git`
5. Commit the changes. The action will run and the results will show in the actions tab on the repository. 
6. Verify the Bitbucket repo - it should now contain a full copy of the github repo including history and branches. 
7. Now any time you push to github the bitbucket repo will also be updated. 