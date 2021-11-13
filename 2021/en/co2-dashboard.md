# Creating a CO2 dashboard for OpenShift: challenges and prospectives

## Motivation

Faced with the climate urgency, everyone needs to do their part. Individuals have an important role to play for sure but companies and their processes can have vastly more leverage when it comes to improving carbon emissions. One aspect of the activity of companies that is often misunderstood and misreported when it comes to carbon emissions is the actual impact of the internet and associated devices, which is estimated to account for up to 3.7% (and rising) of the world's emissions[^1]. While moving to cloud computing shows good promises when it comes to reducing carbon emissions by up to 98%[^2] (!) compared to traditional infrastructure, this actually shifts the emission burden to cloud providers and hyperscalers by "hiding greenhouse gas emissions in the cloud"[^3], moving the emissions from scope 1 or 2 to rarely accounted for scope 3.

There is therefore a need to make the invisible, visible by showing the carbon intensity of applications. At the Red Hat level, the Climate Change Community of Practice, under the lead of Samuel Berthollier and Christophe Laprun, initiated an effort to try to create a CO2 emission dashboard similar to the virtual CPU dashboard already present in the OpenShift console. By showing the carbon intensity of applications running on OpenShift, we could avoid hiding these emissions under the cloud "rug" and raise awareness around the impact of digital as a whole. Measuring / quantifying is the first step towards action. A secondary objective was also to provide a tool that could be used for CSR reporting, provided we could provide accurate enough information. Finally, it could be a nice differentiator compared to our competitors, who, as far as we're aware, don't currently provide such services in their PaaS offering.

## Initial thinking

When we started working on this project, the initial thinking was that we would leverage the information provided by the virtual CPU monitor, which is supposed to measure the CPU utilization incurred by a given application, map this information to power consumption and, from this, map again to associated carbon emissions. We decided early on to focus only on emissions related to CPU usage, though a point could made that these only account for a fraction of the actual emissions. Indeed, to get a full picture one would also need to take into account GPU use, network or storage accesses, etc. We thought, though, that carbon intensity associated with CPU usage would be a first step to start with.

The approach, then, would be to, first retrieve the CPU(s) information (architecture, type, etc.) from the OpenShift instance, compute the power consumption based on the CPU utilization data, retrieve the location of the OpenShift instance on which the application is running and then use the electricityMap API[^5] to map that power consumption into carbon consumption using the carbon intensity information (expressed in grams of CO2 equivalent consumed per kWh of consumed power) provided by the API for that specific location.  

                                                                                                                                                                                                                                
## Challenges

Being complete novices in that field, we ran into several challenges we didn't anticipate.

### CPU power consumption

The first issue we ran into is that there is actually no official data on actual power consumption provided by CPU manufacturers. Indeed, CPU manufacturers usually only provide what is called Thermal Design Power[^6] data, which measures the heat that needs to be dissipated for the CPU to continue functioning under load. While this is somewhat commensurate with actual power consumption, this is not strictly equivalent, in particular because this measure can be manipulated by manufacturers quite easily and is sometimes underestimated, usually for marketing purposes. Another information which would actually be more useful for our purpose is the Average CPU Power (or ACP). However, that information is specific to some CPUs produced by one manufacturer (AMD). 

In short, there is no accurate way to infer power consumption from the virtual CPU information and that CPU's manufacturer's provided information.

### OS-level power measure vs. virtualization
                          
Another area we explored was hardware monitoring softwares such as `hwmon`[^7], `Scaphandre`[^8] which leverages the Intel RAPL[^9] technology. We encountered several issues at that level. The first issue is the fact that while this abstraction layer seems to work reasonably well for Intel-compatible architecture, we would run into issues when dealing with ARM-based architecture, which are more and more popular in server hardware. Another issue is that this assumes access to the underlying OS, which is just not practical (or even impossible) in a virtualized environment.

However, to be useful, our CO2 dashboard would need to work first and foremost on virtualized instances since they account for 90+% of all OCP installations[^4]. The OS-level power measurement was therefore a dead-end for our purposes.
                          
### "Virtualized" power consumption

We then decided to look at cloud providers to see if they would provide the power consumption data but to no avail. Microsoft provides a tool to calculate the emissions associated with workloads running on their Azure cloud offering[^10]. However, that information is not provided at the application level. Microsoft does provide interesting information on the sustainability topic in its blog[^microsoft] and it would be nice to see Red Hat follow suite.

Similarly, Google recently introduced a site[^11] to help its user select cloud regions with low carbon intensity. However, that information is not provided in real-time and it's unclear whether how feasible it is to move workloads to different regions based on carbon intensity.

Amazon, on the other hand, provide some information on how to optimize workloads for "sustainability"[^12] but appears quiet on the carbon front. 

Estimating the carbon impact of cloud applications is something that people are starting to get interested in:
- Benjamin Davy from Teads has a quite detailed set of articles[^teads] trying to estimate the carbon impact of their Amazon workloads which do a good job of showing the complexity of the issue.
- Etsy have also written an article[^etsy] on estimating the carbon impact of their Google Cloud Platform deployments and created an associated tool[^jewels]

### Lack of real-time power mix information

Finally, even if we somehow would have been able to retrieve the instantaneous power consumption for a given application, we would still hit another wall: to accurately compute the associated carbon impact, we would need to know the instantaneous power mix in use by the cloud provider for all the locations where the application runs. That data is also unavailable as we have seen above where the power mix information is provided at best on a per-hour basis and not by every cloud providers.
                           
## Suspension of the initiative

Faced with all these challenges and realizing that getting anywhere would be a full time endeavor, we decided to suspend the project at the beginning of September. However, we do believe that Red Hat / IBM should invest on this front because this problematic is only going to become more pressing as we move forward toward a decarbonized economy. We plan on keeping monitoring the topic and might revive the initiative if the conditions are right to do so.  

## Future work and recommendations

Since we paused the effort, we became aware of several projects that could be helpful for this initiative:

- We became aware of Red Hat's involvement in OS-Climate[^os-climate] and we think it could be a good avenue to try and push cloud providers to be more transparent about their carbon impact. Ideally, a cross-provider API would be provided to measure the carbon impact of deployed applications. Baring this, the next best thing would be to provide access to power consumption and power mix real time data.
- The Cloud Carbon Footprint[^ccf] tool seems very promising and close to our original vision despite being based on estimations instead of accurate measurements. It is an open-source project[^ccf-github] so it might make sense for Red Hatters to get involved.
- Organizatopms such as Climatiq[^climatiq] or WattTime[^watttime] are starting to provide APIs for real-time carbon intensity.
- The PowerAPI[^powerapi] project is an open-source middleware toolkit for building software-defined power meters, which could help on the power consumption measure side of the problem.
- Finally, but not least, we got recently connected to Tamar Eilam, an IBM Fellow working on this very topic at IBM and who's interested in porting her tools to OpenShift!

## Conclusion

While our initiative didn't lead to concrete results, we learned a lot in the process, shedding our initial naïveté regarding the topic of carbon impact of cloud applications. We think that this topic is an important one to address because, as previously mentioned, the impact of cloud technologies is often disregarded because not immediately visible and/or measurable. However, if we want to transition our economy to a fully decarbonized economy, we will need to first quantify the carbon impact of our clouds in order to make them sustainable. We feel that Red Hat is well positioned to be a leader on that topic but we have to admit it's a little disappointing that Red Hat doesn't already appear to be working on similar offerings, especially considering that our efforts to connect with the OpenShift team have mostly been met with silence, in particular when VMWare recently announced an initiative[^vmware] to provide zero-carbon cloud plans to its customers.
                
## Resources
                                                
- [Minutes of the CO2 dashboard workstream](https://docs.google.com/presentation/d/10gM7HatgqQel1l7OyjxzIVOLwe7SGxOZJnc9BYbagg0/edit?pli=1#slide=id.p)
- [Principles of Sustainable Software Engineering](https://principles.green/)
- [Why green cloud optimization is profitable for you and the planet](https://www.thoughtworks.com/insights/articles/green-cloud)
- [How green is your cloud? Tools for reducing your cloud carbon footprint](https://drive.google.com/file/d/1umBCfQXDzz4MJzxjMJE1AQXnb4humkBy/view)

[^1]: https://www.climatecare.org/resources/news/infographic-carbon-footprint-internet/
[^2]: https://www.accenture.com/_acnmedia/PDF-135/Accenture-Strategy-Green-Behind-Cloud-POV.pdf
[^3]: https://www.nature.com/articles/s41558-020-0837-6
[^4]: https://docs.google.com/presentation/d/10UTuNJ2ud5NJscS7vEQ6ixFchTppYxW_1H25JzirV_c/edit#slide=id.gad17f783ee_4_14
[^5]: https://static.electricitymap.org/api/docs/index.html
[^6]: https://en.wikipedia.org/wiki/Thermal_design_power
[^7]: https://www.kernel.org/doc/html/latest/hwmon/hwmon-kernel-api.html
[^8]: https://github.com/hubblo-org/scaphandre
[^9]: https://www.kernel.org/doc/html/latest/power/powercap/powercap.html
[^10]: https://appsource.microsoft.com/en-us/product/power-bi/coi-sustainability.emissions_impact_dashboard
[^microsoft]: https://devblogs.microsoft.com/sustainable-software/
[^11]: https://cloud.google.com/sustainability/region-carbon
[^12]: https://aws.amazon.com/blogs/architecture/optimizing-your-aws-infrastructure-for-sustainability-part-i-compute
[^teads]: https://medium.com/teads-engineering/building-an-aws-ec2-carbon-emissions-dataset-3f0fd76c98ac
[^etsy]: https://codeascraft.com/2020/04/23/cloud-jewels-estimating-kwh-in-the-cloud/
[^jewels]: https://github.com/etsy/cloud-jewels
[^os-climate]: https://os-climate.org/
[^ccf]: https://www.cloudcarbonfootprint.org/
[^ccf-github]: https://github.com/cloud-carbon-footprint/cloud-carbon-footprint
[^climatiq]: https://docs.climatiq.io/
[^watttime]: https://www.watttime.org/api-documentation 
[^vmware]: https://www.sdxcentral.com/articles/news/vmware-zero-carbon-clouds-plan-soars-with-microsoft-ibm/2021/05/