Could one of you look into this request?

Austin Quenneville  |  Senior Engineer Cloud Automation
T +1 813 518 4805   M  +1 813 325 0923
Short Dial *8 115 4805
 
From: Thurman, Jackie <jackie.thurman@whitecase.com>
Date: Thursday, April 23, 2026 at 9:41 AM
To: Quenneville, Austin <austin.quenneville@whitecase.com>; Barrionuevo, Andy <andy.barrionuevo@whitecase.com>; Wray, Dakota <dakota.wray@whitecase.com>
Cc: Garza, Marc <marc.garza@whitecase.com>
Subject: RE: Azure Maps Dependency for SPFx Office Header WebParts
This is the thing that is holding us up for this webpart. Andy had mentioned he wanted the top 4 deployed first before anything else, I have several others waiting to be deployed to prod. If you could pass this off that would be helpful to keep the ball rolling on our end.
Thanks
 
Jackie Thurman  |  Senior Engineer, Applications Development
T +1 813 518 4681     
Short Dial *8 115 4681
 
From: Quenneville, Austin <austin.quenneville@whitecase.com>
Sent: Thursday, April 23, 2026 09:35
To: Thurman, Jackie <jackie.thurman@whitecase.com>; Barrionuevo, Andy <andy.barrionuevo@whitecase.com>; Wray, Dakota <dakota.wray@whitecase.com>
Cc: Garza, Marc <marc.garza@whitecase.com>
Subject: Re: Azure Maps Dependency for SPFx Office Header WebParts
 
What is the timeline for needing this? I won’t have the time personally to address this for a while, but I may be able to get someone else to assist in the meantime.
 
Austin Quenneville  |  Senior Engineer Cloud Automation
T +1 813 518 4805   M  +1 813 325 0923
Short Dial *8 115 4805
 
From: Thurman, Jackie <jackie.thurman@whitecase.com>
Date: Thursday, April 23, 2026 at 9:19 AM
To: Barrionuevo, Andy <andy.barrionuevo@whitecase.com>; Wray, Dakota <dakota.wray@whitecase.com>; Quenneville, Austin <austin.quenneville@whitecase.com>
Cc: Garza, Marc <marc.garza@whitecase.com>
Subject: RE: Azure Maps Dependency for SPFx Office Header WebParts
Morning,
 
Just wanted to follow up on this as this is for one of the original 4 webparts we must get deployed before anything else.  Let me know if you need anything else from us. Thanks
 
Jackie Thurman  |  Senior Engineer, Applications Development
T +1 813 518 4681     
Short Dial *8 115 4681
 
From: Barrionuevo, Andy <andy.barrionuevo@whitecase.com>
Sent: Thursday, April 9, 2026 16:33
To: Wray, Dakota <dakota.wray@whitecase.com>; Quenneville, Austin <austin.quenneville@whitecase.com>
Cc: Thurman, Jackie <jackie.thurman@whitecase.com>; Garza, Marc <marc.garza@whitecase.com>
Subject: RE: Azure Maps Dependency for SPFx Office Header WebParts
 
Hi,
 
                This is a good opportunity to setup Azure subscriptions dedicated for spfx related projects.
                Today, you guys have a need for Azure maps. Tomorrow, it may be something else.
                I added Austin to this thread so he can help with getting the subscriptions and Azure map resource going…
•         Dev – Azure spfx subscription in 3law2law tenant with Azure Maps
•         Stage – Azure spfx subscription in W&C tenant with Azure Maps
•         Prod – Azure spfx subscription in W&C tenant with Azure Maps
 
@Quenneville, Austin – it doesn’t seem to have a huge footprint. What are your thoughts? https://search.opentofu.org/provider/hashicorp/azurerm/latest/docs/resources/maps_account
 
Regards,
Andy
 
Andy Barrionuevo  |  Senior Cloud Engineer, Platform Engineering
T +1 813 518 4718     M +1 813 203 4299    
Short Dial *8 115 4718
 
From: Wray, Dakota <dakota.wray@whitecase.com>
Sent: Tuesday, April 7, 2026 2:26 PM
To: Barrionuevo, Andy <andy.barrionuevo@whitecase.com>
Cc: Thurman, Jackie <jackie.thurman@whitecase.com>; Garza, Marc <marc.garza@whitecase.com>
Subject: Azure Maps Dependency for SPFx Office Header WebParts
 
Hi Andy,
 
The dev/stage version of the https://github.com/wc-application-development/spx-connect-office-header relies on an Azure Maps resource deployed in sub-appdev-dev-primary here azuremaps-dev-eus - Microsoft Azure
This is used for time zone and weather data for offices.
Users (the org-wide W&C groups) need ‘Azure Maps Data Reader’ role assigned on the resource.
 
In package-solution.json we use this permission which was already approved in SP admin.
      {
        "resource": "Azure Maps",
        "scope": "user_impersonation"
      }
To access this via the unified api like…
    const resource = https://atlas.microsoft.com/; // not to be confused with atlas AI
    const client =
      await props.context.aadTokenProviderFactory.getTokenProvider();
    const token = await client.getToken(resource);
 
    const weatherResponse = await fetch(
      `https://atlas.microsoft.com/weather/currentConditions/json?api-version=1.1&query=${props.latitude},${props.longitude}&details=true&language=en-US`,
      {
        headers: {
          Authorization: `Bearer ${token}`,
          "x-ms-client-id": "f72d2253-6ca2-4cc3-9e1e-12941e7317df",
        },
      }
    );


-------------------------------------------------------------------------------------------------------------------
???????????????????????????????????????????????????????????????????????????????????????????????????????????????????
-------------------------------------------------------------------------------------------------------------------
# Add OpenTofu repo
sudo apt-get update
sudo apt-get install -y gnupg software-properties-common curl

curl -fsSL https://packages.opentofu.org/opentofu/tofu/gpgkey | \
  gpg --dearmor -o /usr/share/keyrings/opentofu.gpg

echo "deb [signed-by=/usr/share/keyrings/opentofu.gpg] https://packages.opentofu.org/opentofu/tofu/ubuntu $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/opentofu.list

# Install tofu
sudo apt-get update
sudo apt-get install -y tofu
