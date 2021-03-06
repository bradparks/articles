<sub>&#x1F6A8; <strong>Autogenerated!</strong> See <a href="https://github.com/ponyfoo/articles/tree/noindex/contributing.markdown"><code>contributing.markdown</code></a> for details. See also: <a href="https://ponyfoo.com/articles/modularizing-node-applications-with-express">web version</a>.</sub>

<a href="https://ponyfoo.com/articles/modularizing-node-applications-with-express"><div></div></a>

<h1>Modularizing Node Applications with Express</h1>

<p><kbd>expressjs</kbd> <kbd>nodejs</kbd> <kbd>architecture</kbd></p>

<blockquote><p>I&#x2019;ve spent a few articles talking about <a href="https://ponyfoo.com/search/tagged/build">build processes</a>; now I want to spend a few words on <a href="https://ponyfoo.com/search/tagged/architecture">application architecture</a>, particularly in <em>Node.JS web applications &#x2026;</em></p></blockquote>

<div><p>I&#x2019;ve spent a few articles talking about <a href="https://ponyfoo.com/search/tagged/build">build processes</a>; now I want to spend a few words on <a href="https://ponyfoo.com/search/tagged/architecture">application architecture</a>, particularly in <em>Node.JS web applications using Express</em>.</p></div>

<blockquote></blockquote>

<div><p>In this article, I want to talk about the big picture: <strong>how to separate concerns of applications on different sub-domains in a clean manner</strong>.</p> <p>Sometimes, you need to deal with requests on two separate sub-domains in your application, say <code class="md-code md-code-inline">www</code> and <code class="md-code md-code-inline">blog</code>, but more often than not, this happens within the <em>same express application</em>. While it&#x2019;s <em>certainly possible</em> to handle the routing using a single module, I&#x2019;ve come to realize <strong>it&#x2019;s best to use a modular approach</strong>, similar to what <a href="http://www.senchalabs.org/connect/vhost.html" target="_blank">connect.vhost</a> does, but taken up a notch!</p></div>

<div><figure><a href="http://expressjs.com/" target="_blank" aria-label="Express.JS Web Application Framework"><img alt="express.png" class="" src="https://i.imgur.com/0q7WUxz.png"></a></figure> <p><code class="md-code md-code-inline">connect.vhost</code> simply creates a middleware we can <code class="md-code md-code-inline">.use()</code> in our application, passing it a new server instance for each <em>host</em> we want to handle. With it, we can set up a folder structure that looks <em>somewhat like the following</em>:</p> <pre class="md-code-block"><code class="md-code">src
  - server.js
  + hosts
    + www
      - vhost.js
    + blog
      - vhost.js
</code></pre> <p>With that folder structure in mind, we could code up our <code class="md-code md-code-inline">server.js</code> file using:</p> <pre class="md-code-block"><code class="md-code md-lang-javascript"><span class="md-code-pi">&apos;use strict&apos;</span>;

<span class="md-code-keyword">var</span> express = <span class="md-code-built_in">require</span>(<span class="md-code-string">&apos;express&apos;</span>);
<span class="md-code-keyword">var</span> app = express();
<span class="md-code-keyword">var</span> www = <span class="md-code-built_in">require</span>(<span class="md-code-string">&apos;./hosts/www/vhost.js&apos;</span>);
<span class="md-code-keyword">var</span> blog = <span class="md-code-built_in">require</span>(<span class="md-code-string">&apos;./hosts/blog/vhost.js&apos;</span>);

app.use(www);
app.use(blog);
app.listen(<span class="md-code-number">3000</span>, <span class="md-code-function"><span class="md-code-keyword">function</span><span class="md-code-params">()</span></span>{
    <span class="md-code-built_in">console</span>.log(<span class="md-code-string">&apos;listening on port 3000&apos;</span>);
});
</code></pre> <p>Thus, <code class="md-code md-code-inline">server.js</code> would just contain the logic necessary to enable modularization, and nothing more. Before diving into one of our <code class="md-code md-code-inline">vhost.js</code> files, lets figure out how to build a <em>better <code class="md-code md-code-inline">vhost</code> middleware</em>. Here is what I propose:</p> <pre class="md-code-block"><code class="md-code md-lang-javascript"><span class="md-code-pi">&apos;use strict&apos;</span>;

<span class="md-code-function"><span class="md-code-keyword">function</span> <span class="md-code-title">buildHostRegex</span><span class="md-code-params">(input)</span></span>{
    <span class="md-code-keyword">if</span> (<span class="md-code-keyword">typeof</span> input !== <span class="md-code-string">&apos;string&apos;</span>){
        <span class="md-code-keyword">return</span> input;  
    }
    
    <span class="md-code-keyword">var</span> hostname = input || <span class="md-code-string">&apos;*&apos;</span>;
    <span class="md-code-keyword">if</span> (hostname.test(<span class="md-code-regexp">/[^A-z_0-9.*-]/</span>)){
        <span class="md-code-keyword">throw</span> <span class="md-code-keyword">new</span> <span class="md-code-built_in">Error</span>(<span class="md-code-string">&apos;Invalid characters in host pattern: &apos;</span> + hostname);
    }

    <span class="md-code-keyword">var</span> pattern = hostname
        .replace(<span class="md-code-regexp">/\./g</span>, <span class="md-code-string">&apos;\\.&apos;</span>)
        .replace(<span class="md-code-regexp">/\*/g</span>,<span class="md-code-string">&apos;(.*)&apos;</span>);

    <span class="md-code-keyword">var</span> rhost = <span class="md-code-keyword">new</span> <span class="md-code-built_in">RegExp</span>(<span class="md-code-string">&apos;^&apos;</span> + pattern + <span class="md-code-string">&apos;$&apos;</span>, <span class="md-code-string">&apos;i&apos;</span>);
    <span class="md-code-keyword">return</span> rhost;
}

<span class="md-code-built_in">module</span>.exports = <span class="md-code-function"><span class="md-code-keyword">function</span><span class="md-code-params">(factory)</span></span>{
    <span class="md-code-keyword">return</span> <span class="md-code-function"><span class="md-code-keyword">function</span><span class="md-code-params">(hostname)</span></span>{
        <span class="md-code-keyword">var</span> rhost = buildHostRegex(hostname);

        <span class="md-code-keyword">var</span> middleware = <span class="md-code-function"><span class="md-code-keyword">function</span><span class="md-code-params">(req, res, next)</span></span>{
            <span class="md-code-keyword">if</span>(middleware.on === <span class="md-code-literal">false</span>){
                <span class="md-code-keyword">return</span> next();
            }

            <span class="md-code-keyword">if</span>(!req.headers.host){
                <span class="md-code-keyword">return</span> next();
            }

            <span class="md-code-keyword">var</span> host = req.headers.host.split(<span class="md-code-string">&apos;:&apos;</span>)[<span class="md-code-number">0</span>];
            <span class="md-code-keyword">if</span>(!rhost.test(host)){
                <span class="md-code-keyword">return</span> next();
            }

            <span class="md-code-keyword">if</span> (<span class="md-code-keyword">typeof</span> middleware.server === <span class="md-code-string">&apos;function&apos;</span>){
                <span class="md-code-keyword">return</span> middleware.server(req, res, next);
            }
            middleware.server.emit(<span class="md-code-string">&apos;request&apos;</span>, req, res);
        };

        middleware.server = factory();

        middleware.off = <span class="md-code-function"><span class="md-code-keyword">function</span><span class="md-code-params">()</span></span>{
            middleware.on = <span class="md-code-literal">false</span>;
        };

        middleware.on = <span class="md-code-function"><span class="md-code-keyword">function</span><span class="md-code-params">()</span></span>{
            middleware.on = <span class="md-code-literal">true</span>;
        };

        <span class="md-code-keyword">return</span> middleware;
    };
};
</code></pre> <p>Using <a href="https://github.com/bevacqua/virtual-host" target="_blank" aria-label="virtual-host on GitHub">this module</a>, this separation of concerns becomes almost trivial.</p> <p>Our <code class="md-code md-code-inline">./www/vhost.js</code> implementation can become:</p> <pre class="md-code-block"><code class="md-code md-lang-javascript"><span class="md-code-pi">&apos;use strict&apos;</span>;

<span class="md-code-keyword">var</span> express = <span class="md-code-built_in">require</span>(<span class="md-code-string">&apos;express&apos;</span>);
<span class="md-code-keyword">var</span> vhost = <span class="md-code-built_in">require</span>(<span class="md-code-string">&apos;virtual-host&apos;</span>)(express);
<span class="md-code-keyword">var</span> www = vhost(<span class="md-code-string">&apos;www.*&apos;</span>);
<span class="md-code-keyword">var</span> app = www.server;

app.use(<span class="md-code-function"><span class="md-code-keyword">function</span><span class="md-code-params">(req,res,next)</span></span>{
    res.send(<span class="md-code-string">&apos;response from www!&apos;</span>);
});

<span class="md-code-built_in">module</span>.exports = www;
</code></pre> <p>The important factor in this separation is that we can handle <em>views, static assets, routing, and pretty much every aspect</em> of our application in a self-contained manner. We could always <em>re-use common pieces of functionality</em> through <code class="md-code md-code-inline">require</code>, as per usual.</p> <p>I can think of a few use cases for this sort of modularization</p> <ul> <li>Setting up a sub-domain for <strong>API documentation, blog, or custom solution</strong>, while <em>keeping our main application unaltered</em>, when resorting to a third-party solution for one of our sub-domains</li> <li>Setting up a request catch-all which is great for displaying a <strong>different application altogether</strong> for the initial setup of open-source projects, <em>helping developers</em> to set up their local environments</li> <li>Setting up a maintenance catch-all which can be turned on and off as a service, allowing us to have an <strong>emergency off switch</strong> without necessarily turning off job queues, and allowing for a more responsive alternative to turn the server back online</li> </ul> <p>How might the maintenance setup work? Probably something close to this:</p> <pre class="md-code-block"><code class="md-code md-lang-javascript"><span class="md-code-pi">&apos;use strict&apos;</span>;

<span class="md-code-keyword">var</span> express = <span class="md-code-built_in">require</span>(<span class="md-code-string">&apos;express&apos;</span>);
<span class="md-code-keyword">var</span> app = express();
<span class="md-code-keyword">var</span> www = <span class="md-code-built_in">require</span>(<span class="md-code-string">&apos;./hosts/www/vhost.js&apos;</span>);
<span class="md-code-keyword">var</span> maintenance = <span class="md-code-built_in">require</span>(<span class="md-code-string">&apos;./hosts/maintenance/vhost.js&apos;</span>);

app.use(maintenance);
app.use(www);
app.listen(<span class="md-code-number">3000</span>, <span class="md-code-function"><span class="md-code-keyword">function</span><span class="md-code-params">()</span></span>{
    <span class="md-code-built_in">console</span>.log(<span class="md-code-string">&apos;listening on port 3000&apos;</span>);
});
</code></pre> <p>In the maintenance <code class="md-code md-code-inline">vhost.js</code>:</p> <pre class="md-code-block"><code class="md-code md-lang-javascript"><span class="md-code-pi">&apos;use strict&apos;</span>;

<span class="md-code-keyword">var</span> express = <span class="md-code-built_in">require</span>(<span class="md-code-string">&apos;express&apos;</span>);
<span class="md-code-keyword">var</span> vhost = <span class="md-code-built_in">require</span>(<span class="md-code-string">&apos;virtual-host&apos;</span>)(express);
<span class="md-code-keyword">var</span> maintenance = vhost(<span class="md-code-string">&apos;*&apos;</span>); <span class="md-code-comment">// catch-all</span>
<span class="md-code-keyword">var</span> app = maintenance.server;

app.use(<span class="md-code-function"><span class="md-code-keyword">function</span><span class="md-code-params">(req,res,next)</span></span>{
    res.send(<span class="md-code-string">&apos;the server is in maintenance mode!&apos;</span>);
});

maintenance.off(); <span class="md-code-comment">// disabled by default</span>

<span class="md-code-built_in">module</span>.exports = maintenance;
</code></pre> <p>We might have a <em>background job</em> that allows us to <strong>turn the maintenance vhost on or off</strong>, and that would be it. What&#x2019;s even better, it would be <em>reusable</em> in any applications we craft following this pattern.</p> <p>If you want to read <em>more articles like this</em>, follow the <a href="https://ponyfoo.com/rss/latest.xml" aria-label="RSS feed for this blog">RSS feed</a>, or subscribe to the email updates!</p></div>
