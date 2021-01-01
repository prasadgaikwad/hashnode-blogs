## How I created my blog in 10 minutes!


> Note: The steps mentioned here are specifically for Google DNS, a .dev domain and Hashnode.

So I wanted to create my blog and online profile for quite some time but was not sure about following:

1. How to start the blog?
2. How to register a domain name and setup?
3. Should I use any blogging platform or build my own blog?
4. Which blogging platform to use?

And after researching for a while, I decided to create my blog today to check this item off my list on very first day of the year :)

Here is a short tutorial on how to create a blog, setup your own domain name and use the domain name to redirect to your blog, all about in 10 minutes. (off course it took some time for me to research on the options for blogging platforms, domain name service providers etc, but actual process of blog setup took about 10 minutes!)

# My Choices

- I chose [Google Domains](https://domains.google.com/) as my domain name service (DNS) provider to avoid getting too much into the weeds for searching a DNS provider. During my initial research, I found it has features which I required to get started quickly. 
- I picked [Hashnode](https://hashnode.com/) as my blogging platform since I found that it provides a lot of useful features in addition to the ease of creating and maintaining a blog.

# Prerequisites

1. You need a google/gmail account to buy domain name from google.
2. Create an account with Hashnode to get you started (you don't need steps for this, believe me ;)) 

# Steps

1. Search for your required domain name at [Google Domains](https://domains.google.com/), add it to the cart and proceed.
You should see something like following on next screen:
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1609529870646/WLq97Uwyg.png)
2. Provide required contact information on checkout screen. 
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1609530136994/ZtlCq1VK2.png)
3. Provide payment information and checkout. That's it! You own the domain name now! You may have to verify your email address for the domain purchase.
4. On your Hashnode blog dashboard page, add your domain name (in my case I used a subdomain blog.prasadgaikwad.dev) under DOMAIN tab and click update. You will see next steps on the same page as shown below:
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1609530572227/rxmEuzi2WJ.png)
5. Go back to your Google Domains dashboard and select the newly created domain name to go into its settings.
6. Go to DNS section and scroll to the "Custom resource records" section. Add CNAME record as shown below:
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1609530956828/y2Idz3sBS.png)
(Note that I am using a subdomain "blog" since Google Domains does not support CNAME record for root domain. You can create any subdomain)
7. Wait for few minutes and go to your subdomain url, you should see your Hashnode blog!
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1609531345453/e9u7ckSJo.png)
8. Wait for it...you may also want to redirect your root domain (in my case  [prasadgaikwad.dev](prasadgaikwad.dev)  and  [www.prasadgaikwad.dev](www.prasadgaikwad.dev)), then go the Website section of your Google Domains page and add Forward domain details. (In my case I forwarded my root domain to my blog subdomain for now!)
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1609531593479/FKspWiQib.png)
9. You may have to hard refresh (command + r) browser page with your domain url, to delete any cached pages or errors.
10. And that's all you need to get your own developer blog started on your custom domain.

# References:
1.  [How to Set Up a Custom Domain on Devblog](https://hashnode.com/post/how-to-set-up-a-custom-domain-on-devblog-cjvoymax9001u8xs1i9f3077e) 
2.  [Hashnode: How to Launch Your Own Developer Blog on Your Own Domain in Minutes](https://www.freecodecamp.org/news/devblog-launch-your-developer-blog-own-domain/) 

(I created this blog post to document steps which I could not find in reference blogs, so that it might be useful if someone decides to use these platforms like me) 

