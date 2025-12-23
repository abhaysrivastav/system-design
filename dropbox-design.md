# Design Dropbox - System Design Case Study


## User should be able to share the file with other user 

user1: [file1, file2, file3]
user2: [file1, file2, file3]

shareList is in file metadata
sharedFiles is the cache for users

Challenge: 

shareList and sharedFiles should be consistent

The best way to solve this problem that shareList and sharedFiles are consistent should be work in transaction.


Great Solution : Create a new table called share 


## User Can automatically sync files across devices.

We need to sync in 2 directions:

1. User can add files to their local machine and it should sync to the cloud.
2. If and file is updated in the cloud, it should sync to the local machine.


### Local--->Cloud

- We need a client side agent to sync files to the cloud.

### Cloud--->Local

- There are 2 possible approaches 

1. Polling
2. Websocket: It will maintain a open connection between client and server, and pushes the notification when changes occurs. This is complex but provide the real time experience.

For dropbox, we can use hybrid approach.
- For Fresh files : We can use websockets , to prode the real time sync.
- For Old files : We can use polling.






