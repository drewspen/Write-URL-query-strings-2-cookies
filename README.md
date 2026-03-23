# Write-URL-query-strings-2-cookies
Write URL query strings 2 cookies

Years ago, I needed to pass landing page URL query string parameters into a CRM three pages deep into the customer journey. Inspired by an Analytics Mania post on link decoration, I developed the first version of this solution.

The landing URL query string parameters were written to root domain cookies. A page URL could be decorated, or hidden CRM form fields populated, elsewhere on the site or subdomains. 

This latest incarnation is a Good Tag Manager Tag template utilizing Sandboxed JavaScript to not require any additional CSP (Content Security Policy) rules or SHA256 keys. It has the following features.

1) The root domain is a parameters in the event you wish to use a subdomain instead. 
2) You can choose between persistent and session cookies to allow for the measurement to be visitor or visit level. 
3) You can add a suffix to the cookie name in the event persistent and session measurements must both exist. 
4) You can choose to URI decode the URL query string.
5) Although a long list of preselected URL query string parameters are available, you can deselect the unwanted and add new ones not listed.
6) If the cookies already exist they are not overwritten. If the URL query string parameters do not exist they are not written to cookies.

From here I can decorate the cookie name and value pairs onto iframe src (etc), or into hidden fields of a page form.
