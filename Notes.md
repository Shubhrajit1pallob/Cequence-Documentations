- After STS validation, clear process.env.AWS_* so the next account's prompt doesn't accidentally reuse stale env vars — important for the multi-account loop.


- What are Nginx directives?
-  The list of the CIDR ranges for the edge cases.



## Plat-981
- Think about how we could improve this
- How could we automate this entirely (from design to execution) where we are just waiting for the output in an MR to review
- How can you verify this as a human? 
- Btw, as long as you run in dry run mode, you can run ops portal to deploy and upgrade fake tenants.  It is possible and no real calls are made. All commits end up being local on your computer.