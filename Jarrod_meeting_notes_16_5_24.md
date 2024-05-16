# Summary
	- I know what the issue is with our original approach
	- In the meeting I feel I understood the multinomial model and how to fit it/what it meant
	- Since the meeting I think I have managed to forget everything and am now fairly confused so below I have outlined what I think happened and I am very sorry it might not make a lot of sense/ just be outright wrong.
- # Multinomial models
	- the high correlation we are observing between indel vs snp diveristy in our plots:
		- ![Screenshot 2024-05-16 at 11.08.01.png](../assets/Screenshot_2024-05-16_at_11.08.01_1715854083171_0.png)
	- Is most likely just due to the fact that the denominator of our equations:
		- $$\pi_{SNP} = \frac{No.\ of\ snps}{No.\ aligned\ sites}$$
		- $$\pi_{indel} = \frac{No.\ of\ indels}{No.\ aligned\ sites + \frac{No.\ bp\ in\ indels}{2}}$$
		- Are very similar and so therefore they are driving our observed correlation!
			- (autosomes in above graph: $$R^2 = 0.86$$ and $$P\ value = < 2.2 e^{-16}$$)
			- Jarrod = suspicious when $$R^2 > 0.5$$ as this is often due to both axes having the same term in them (or very close to same term in our case)
	- SO! To get around this Jarrod (JH from now on) suggests that we use a multinomial method for this analysis.
		- essentially we split all the sites in the genome (that we can align) into 3 subsets:
			- monomorphic
			- SNP
			- Indel
		- We then do log(odds ratios) in order to determine the log of the odds that any site in the genome is a SNP or Indel site compared to a monomorphic site:
			- Example where we have 100 monomorphic sites, 50 SNP sites and 75 indel sites:
				- odds ratio for SNPs:
					- $$50 : 100 = \frac{1}{2}$$
				- odds ratio for indels:
					- $$75:100 = \frac{3}{4}$$
				- You then plot these values logged to get logged odds
				- ![Screenshot 2024-05-16 at 11.26.06.png](../assets/Screenshot_2024-05-16_at_11.26.06_1715855169580_0.png)
					- I think i'm still a bit confused about how we visualise this model? (I made up some other values for the above graph)
						- ![Screenshot 2024-05-16 at 11.35.59.png](../assets/Screenshot_2024-05-16_at_11.35.59_1715855763216_0.png)
					- So this would be telling us about the relationship between the probability that a site is a SNP compared to being monomorphic vs the probability that a site is an indel compared to being monomorphic.
						- expectation:
							- Higher diversity species should have a higher probability that a Site is a SNP or indel as the fact that they are higher diversity means that they have more SNPs or indels than lower diversity species. So still answering the same question in a way, the probability that sites are indel or SNP is correlated to the amount of diversity present in the genome but this just allows us to have graph axes that don't have the same terms in them?
							- **HOWEVER**
								- don't they kind of have the same axes?
								- $$ X = \text{log}(\frac{no.\ SNP\ sites}{no.\ monomorphic\ sites})$$
								- vs
								- $$Y= \text{log}(\frac{no.\ indel\ sites}{no.\ monomorphic\ sites})$$
								- So they both have monomorphic sites on the denominator again?
								- I think it is okay as it is something to do with you've turned it into a probability?
					- There was also a cool thing about how using this method allowed for us to reduce the amount of within-species variation that we get in the residuals meaning that the remaining residuals should mainly be the between species variance that we are then going to see how much of is due to phylogeny with MCMCglmm.
						- $$\pi = \text{something} + e_{\text{within and between species}}$$
						- where e is our residual, pi is probability and something is our model
						- $$logit(\pi) = \text{something} + e_{\text{between species}}$$
							- ????
		- Potential issue with this is that:
			- non 1 bp indels occur, this will impact the number of bp that have a probability of being an indel. Particularly for species that have large number of indels - need to think about how this will effect the analysis!
		- fitting this model with mcmcglmm:
			- the main point is:
				- ```
				  MCMCglmm(cbind(1,2,3) ~ trait -1,
				          rcov=~trait:units
				          family='multinomial3'
				          pl=TRUE
				          [rest of model as normal]
				  ```
			- trait 3 will need to be monomorphic sites so that becomes the baseline that we compare to
				- could be any of them and it will work but using monomorphic sites makes the most intuitive sense
			- rcov - allows us to say there is covariance between the traits (SNP, indel or monomorphic)?
				- | |SNP|INDEL|MONOMORPHIC|
				  |SNP|\sigma^{2}_{SNP}|\sigma_{SNP,INDEL}|\sigma_{SNP,MONOMORPHIC}|
				  |INDEL|\sigma_{INDEL,SNP}|\sigma^{2}_{INDEL}|\sigma_{INDEL,MONOMORPHIC}|
				  |MONOMORPHIC|\sigma_{MONOMORPHIC,SNP}|\sigma_{MONOMORPHIC,INDEL}|\sigma^{2}_{MONOMORPHIC}|
			- pl=TRUE allows the random effect distributions to also be stored and then we can use that to plot our outputs???
	- There was another method that wasn't as nice as the multinomial model where we just take logs of the original equations?
		- might be remembering this wrong
- # Priors
	- ```
	  prior1<-list(G=list(G1=list(V=diag(3),nu=3, alpha.mu=c(0,0), alpha.V=diag(3)*1000)), 
	  R=list(V=diag(2),nu=0.002))
	  
	  ```
	- These are the priors we will go for.
	- By using the expanded priors this creates the *unknown* prior distribution. Not much known about it but by adding in alpha.mu and alpha.V this adds in parameters that aren't present in the data and somehow this makes the model better at determining the values for the parameters that are present in the data?
	- We set the V to 1 in the diagonals, three diagonals for 3 variables?
	- nu is the belif parameter
		- in the model with just indel ~ snp it was set to 1. In Alex's model (with 2 response variables he had 2) so for ours we set 3 in this case? - complete guess apologies
	- alpha.mu - one of our 'fake' parameters
	- alpha.v - the other of our 'fake' parameters, we set our variance to a nice big number so we can move around more parameter space?
	- By setting our parameters in this way, this makes our prior flat which people think is good as they belive that therefore it is only their data that will influence the model as the prior has no variance?
		- However it is not truely flat, priors are parameters that have to equal 1. So while yes the point we are looking at is flat, eventually it will trend downwards and you'll end up with:
			- ![image.png](../assets/image_1715864655854_0.png)
			- This is important to remember.
- # Log 4D model:
	- ![Screenshot 2024-05-16 at 14.06.55.png](../assets/Screenshot_2024-05-16_at_14.06.55_1715864820425_0.png)
	- Okay so we have a similar issue here to the one outlined above where we have the same thing on both axes.
	- the way to fit this would be to have a 4 way multinomial model where we have SNPs broken down into 4D SNPs, 0D SNPs, monomorphic sites and indel sites.
		- (there are the other snp sites that aren't 0 or 4 d that aren't included in this analysis but i'm guessing we're only focussed on the ones we're interested in)
	- Again I get how to fit the mcmcglmm model (i think)
		- I think my understanding of the log odds ratio is just a little shaky, so i'm going to understand the first part then come back to this
- Again sorry! You did explain it really well it is just a little twisted in my head