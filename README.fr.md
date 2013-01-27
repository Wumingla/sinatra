# Sinatra
*Attention : Ce document correspond à la traduction de la version anglaise et
il n'est peut être plus à jour.*

Sinatra est un [DSL](http://fr.wikipedia.org/wiki/Langage_dédié) pour
créer rapidement et facilement des applications web en Ruby :

```ruby
# mon_application.rb
require 'sinatra'

get '/' do
  'Bonjour le monde !'
end
```

Installez la gem et lancez avec :

```bash
$ gem install sinatra
$ ruby mon_application.rb
```

Le résultat est visible sur : http://localhost:4567

Il est recommandé d'exécuter également `gem install thin`, pour que
Sinatra utilise le server Thin quand il est disponible.

## Routes

Dans Sinatra, une route est une méthode HTTP couplée à un masque (pattern)
URL. Chaque route est associée à un bloc :

```ruby
get '/' do
  .. montrer quelque chose ..
end

post '/' do
  .. créer quelque chose ..
end

put '/' do
  .. remplacer quelque chose ..
end

patch '/' do
  .. changer quelque chose ..
end

delete '/' do
  .. effacer quelque chose ..
end

options '/' do
  .. apaiser quelquechose ..
end
```

Les routes sont évaluées  dans l'ordre où elles ont été définies. La première
route qui correspond à la requête est appelée.

Les masques peuvent inclure des paramètres nommés, accessibles par
l'intermédiaire du hash `params` :

```ruby
get '/bonjour/:nom' do
  # répond aux requêtes "GET /bonjour/foo" et "GET /bonjour/bar"
  # params[:nom] est 'foo' ou 'bar'
  "Bonjour #{params[:nom]} !"
end
```

Vous pouvez aussi accéder aux paramètres nommés directement grâce aux
paramètres du bloc comme ceci :

```ruby
get '/bonjour/:nom' do |n|
  "Bonjour #{n} !"
end
```

Une route peut contenir un splat (caractère joker), accessible par
l'intermédiaire du tableau `params[:splat]` :

```ruby
get '/dire/*/a/*' do
  # répond à /dire/bonjour/a/monde
  params[:splat] # => ["bonjour", "monde"]
end

get '/telecharger/*.*' do
  # répond à /telecharger/chemin/vers/fichier.xml
  params[:splat] # => ["chemin/vers/fichier", "xml"]
end
```

Ou par l'intermédiaire des paramètres du bloc :

```ruby
get '/telecharger/*.*' do |chemin, ext|
  [chemin, ext] # => ["path/to/file", "xml"]
end
```

Une route peut aussi être définie par une expression régulière :

```ruby
get %r{/bonjour/([\w]+)} do
  "Bonjour, #{params[:captures].first} !"
end
```

Là encore on peut utiliser les paramètres de bloc :

```ruby
get %r{/bonjour/([\w]+)} do |c|
  "Bonjour, #{c} !"
end
```

Les routes peuvent aussi comporter des paramètres optionnels :

```ruby
get '/posts.?:format?' do
  # répond à "GET /posts" et aussi à "GET /posts.json", "GET /posts.xml" etc...
end
```

A ce propos, à moins d'avoir désactivé la protection contre les attaques par
"path transversal" (voir plus loin), l'URL demandée peut avoir été modifiée
avant d'être comparée à vos routes.

### Conditions

Les routes peuvent définir toutes sortes de conditions, comme par exemple le
"user agent" :

```ruby
get '/foo', :agent => /Songbird (\d\.\d)[\d\/]*?/ do
  "Vous utilisez Songbird version #{params[:agent][0]}"
end

get '/foo' do
  # Correspond à tous les autres navigateurs
end
```

Les autres conditions disponibles sont `host_name` et `provides` :

```ruby
get '/', :host_name => /^admin\./ do
  "Zone Administrateur, Accès refusé !"
end

get '/', :provides => 'html' do
  haml :index
end

get '/', :provides => ['rss', 'atom', 'xml'] do
  builder :feed
end
```

Vous pouvez facilement définir vos propres conditions :

```ruby
set(:probability) { |value| condition { rand <= value } }

get '/gagner_une_voiture', :probability => 0.1 do
  "Vous avez gagné !"
end

get '/gagner_une_voiture' do
  "Désolé, vous avez perdu."
end
```

Utilisez un splat (caractère joker) dans le cas d'une condition qui prend
plusieurs valeurs :

```ruby
set(:auth) do |*roles|   # <- ici on utilise un splat
  condition do
    unless logged_in? && roles.any? {|role| current_user.in_role? role }
      redirect "/login/", 303
    end
  end
end

get "/mon/compte/", :auth => [:user, :admin] do
  "Informations sur votre compte"
end

get "/reserve/aux/admins/", :auth => :admin do
  "Seuls les administrateurs sont acceptés ici !"
end
```

### Valeurs de retour

La valeur renvoyée par le bloc correspondant à une route constitue le corps de
la réponse qui sera transmise au client HTTP ou du moins au prochain middleware
dans la pile Rack. Le plus souvent, il s'agit d'une chaîne de caractères,
comme dans les exemples précédents. Cependant, d'autres valeurs sont
acceptées.

Vous pouvez renvoyer n'importe quel objet qu'il s'agisse d'une réponse Rack
valide, d'un corps de réponse Rack ou d'un code statut HTTP :

* Un tableau de 3 éléments : `[code statut (Fixnum), entêtes (Hash), corps
  de la réponse (répondant à #each)]`
* Un tableau de 2 élements : `[code statut (Fixnum), corps de la réponse
  (répondant à #each)]`
* Un objet qui répond à `#each` et qui ne transmet que des chaînes de
  caractères au bloc fourni
* Un Fixnum représentant le code statut

Avec cela, on peut facilement implémenter un streaming par exemple :

```ruby
class Stream
  def each
    100.times { |i| yield "#{i}\n" }
  end
end

get('/') { Stream.new }
```

Vous pouvez aussi utiliser le helper `stream` (présenté un peu plus loin) pour
éviter la surcharge et intégrer le traitement relatif au streaming dans le bloc
de code de la route.

### Masques de route spécifiques

Comme cela a été vu auparavant, Sinatra offre la possibilité d'utiliser des
masques sous forme de chaines de caractères ou des expressions régulières
pour définir les routes. Mais il est possible de faire bien plus. Vous pouvez
facilement définir vos propres masques :

```ruby
class MasqueToutSauf
  Masque = Struct.new(:captures)

  def initialize(except)
    @except   = except
    @captures = Masque.new([])
  end

  def match(str)
    @caputres unless @except === str
  end
end

def tout_sauf(masque)
  MasqueToutSauf.new(masque)
end

get tout_sauf("/index") do
  # ...
end
```

Notez que l'exemple ci-dessus est bien trop compliqué et que le même résultat
peut être obtenu avec :

```ruby
get // do
  pass if request.path_info == "/index"
  # ...
end
```

Ou bien en utilisant la forme négative :

```ruby
get %r{^(?!/index$)} do
  # ...
end
```

## Fichiers statiques

Les fichiers du dossier `./public` sont servis de façon statique. Vous
avez la possibilité d'utiliser un autre répertoire en définissant le paramètre
`:public_folder` :

```ruby
set :public_folder, File.dirname(__FILE__) + '/statique'
```

Notez que le nom du dossier public n'apparait pas dans l'URL. Le fichier
`./public/css/style.css` sera appelé via l'URL :
`http://exemple.com/css/style.css`.

Utilisez le paramètre `:static_cache_control` pour ajouter l'information
d'en-tête <tt>Cache-Control</tt> (voir plus loin).

## Vues / Templates

Chaqie langage de template est disponible via sa propre méthode de rendu,
lesquelles renvoient tout simplement une chaîne de caractères.

```ruby
get '/' do
  erb :index
end
```

Ceci effectue le rendu de la vue `views/index.erb`.

Plutôt que d'utiliser le nom d'un template, vous pouvez directement passer
le contenu du template :

```ruby
get '/' do
  code = "<%= Time.now %>"
  erb code
end
```

Les méthodes de templates acceptent un second paramètre, un hash d'options :

```ruby
get '/' do
  erb :index, :layout => :post
end
```

Ceci effectuera le rendu de la vue `views/index.erb` en l'intégrant
au *layout* `views/post.erb` (les vues Erb sont intégrées par défaut
au *layout* `views/layout.erb` quand ce fichier existe).

Toute option que Sinatra ne comprend pas sera passée au moteur de rendu :

```ruby
get '/' do
  haml :index, :format => :html5
end
```

Vous pouvez également définir des options par langage de template de façon
générale :

```ruby
set :haml, :format => html5

get '/' do
  haml :index
end
```

Les options passées à la méthode de rendu prennent le pas sur les options
définies au moyen de `set`.

Options disponibles :

**locals**
  Liste de variables locales passées au document. Pratique pour les vues
  partielles.
  Exemple : `erb "<%= foo %>", :locals => {:foo => "bar"}`.

**default_encoding**
  Encodage de caractères à utiliser en cas d'incertitude. Par défaut, c'est
  `settings.default_encoding`.

**views**
  Dossier de vues dans lequel chercher les templates. Par défaut
  `settings.views`.

**layout**
  S'il faut ou non utiliser un +layout+ (+true+ or +false+). Indique le
  template à utiliser lorsque c'est un symbole. Exemple : `erb :index,
  :layout => !request.xhr?`.

**content_type**
  Content-Type que le template produit, dépend par défaut du langage de
  template.

**scope**
  Contexte sous lequel effectuer le rendu du template. Par défaut il s'agit
  de l'instance de l'application. Si vous changez cela, les variables
  d'instance et les méthodes utilitaires ne seront pas disponibles.

**layout_engine**
  Moteur de rendu à utiliser pour le +layout+. Utile pour les langages ne
  supportant pas les +layouts+. Il s'agit par défaut du moteur utilisé pour
  le rendu du template. Exemple : `set :rdoc, :layout_engine => :erb`

Les templates sont supposés se trouver directement dans le dossier
`./views`. Pour utiliser un dossier de vues différent :

```ruby
set :views, settings.root + '/templates'
```

Il est important de se souvenir que les templates sont toujours référencés
sous forme de symboles, même lorsqu'ils sont dans un sous-répertoire (dans
ce cas, utilisez `:'sous_repertoire/template'`). Il faut utiliser
un symbole car les méthodes de rendu évaluent le contenu des chaînes de
caractères au lieu de les considérer comme un chemin vers un fichier.

### Langages de template disponibles

Certains langages ont plusieurs implémentations. Pour préciser l'implémentation
à utiliser (et garantir l'aspect thread-safe), vous devez simplement l'avoir
chargée au préalable :

```ruby
require 'rdiscount' # ou require 'bluecloth'
get('/') { markdown :index }
```

### Templates Haml

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://haml.info/">haml</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.haml</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>haml :index, :format => :html5</tt></td>
  </tr>
</table>

### Templates Erb

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://www.kuwata-lab.com/erubis/">erubis</a> ou erb (inclus avec Ruby)</td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.erb</tt>, <tt>.rhtml</tt> ou <tt>.erubis</tt> (Erubis seulement)</td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>erb :index</tt></td>
  </tr>
</table>

### Templates Builder

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://builder.rubyforge.org/">builder</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.builder</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>builder { |xml| xml.em "salut" }</tt></td>
  </tr>
</table>

Ce moteur accepte également un bloc pour des templates en ligne (voir
exemple).

### Templates Nokogiri

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://nokogiri.org/">nokogiri</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.nokogiri</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>nokogiri { |xml| xml.em "salut" }</p>
    </td>
  </tr>
</table>

Ce moteur accepte également un bloc pour des templates en ligne (voir
exemple).

### Templates Sass

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://sass-lang.com/">sass</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.sass</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>sass :stylesheet, :style => :expanded</tt></td>
  </tr>
</table>

### Templates SCSS

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://sass-lang.com/">sass</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.scss</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>scss :stylesheet, :style => :expanded</tt></p>
    </td>
  </tr>
</table>

### Templates Less

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://www.lesscss.org/">less</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.less</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>less :stylesheet</tt>
    </td>
  </tr>
</table>

### Templates Liquid

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://www.liquidmarkup.org/">liquid</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.liquid</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>liquid :index, :locals => { :key => 'value' }</tt></td>
  </tr>
</table>

Comme vous ne pouvez appeler de méthodes Ruby (autres que `yield`)
dans un template Liquid, vous aurez sûrement à lui passer des variables
locales.

### Templates Markdown

<table>
  <tr>
    <td><p>Dépendances</p></td>
    <td><a href="https://github.com/rtomayko/rdiscount">rdiscount</a>, <a href="https://github.com/vmg/redcarpet">redcarpet</a>, <a href="http://deveiate.org/projects/BlueCloth">bluecloth</a>, <a href="http://kramdown.rubyforge.org/">kramdown</a> *ou* <a href="http://maruku.rubyforge.org/">maruku</a></td>
  </tr>

  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.markdown</tt>, <tt>.mkd</tt> et <tt>.md</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>markdown :index, :layout_engine => :erb</tt></td>
  </tr>
</table>

Il n’est pas possible d’appeler des méthodes depuis markdown, ni de
lui passer des variables locales. Par conséquent, il sera souvent utilisé
en combinaison avec un autre moteur de rendu :

```ruby
erb :overview, :locals => { :text => markdown(:introduction) }
```

Notez que vous pouvez également appeler la méthode `markdown` au
sein d’autres templates :

```ruby
%h1 Hello From Haml !
%p= markdown(:greetings)
```

Comme vous ne pouvez pas appeler de Ruby au sein de Markdown, vous ne
pouvez pas utiliser de layouts écrits en Markdown. Toutefois, il
est possible d’utiliser un moteur de rendu différent pour le template et
pour le layout en utilisant l’option `:layout_engine`.

### Templates Textile

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://redcloth.org/">RedCloth</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.textile</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>textile :index, :layout_engine => :erb</tt></td>
  </tr>
</table>

Il n’est pas possible d’appeler des méthodes depuis textile, ni de lui
passer des variables locales. Par conséquent, il sera souvent utilisé en
combinaison avec un autre moteur de rendu :

```ruby
erb :overview, :locals => { :text => textile(:introduction) }
```

Notez que vous pouvez également appeler la méthode `textile` au
sein d’autres templates :

```ruby
%h1 Hello From Haml !
%p= textile(:greetings)
```

Comme vous ne pouvez pas appeler de Ruby au sein de Textile, vous ne pouvez
pas utiliser de layouts écrits en Textile. Toutefois, il est
possible d’utiliser un moteur de rendu différent pour le template et
pour le layout en utilisant l’option `:layout_engine`.

### Templates RDoc

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://rdoc.rubyforge.org/">rdoc</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.rdoc</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>rdoc :README, :layout_engine => :erb</tt></td>
  </tr>
</table>

Il n’est pas possible d’appeler des méthodes depuis rdoc, ni de lui
passer des variables locales. Par conséquent, il sera souvent utilisé en
combinaison avec un autre moteur de rendu :

```ruby
erb :overview, :locals => { :text => rdoc(:introduction) }
```

Notez que vous pouvez également appeler la méthode `rdoc` au sein
d’autres templates :

```ruby
%h1 Hello From Haml !
%p= rdoc(:greetings)
```

Comme vous ne pouvez pas appeler de Ruby au sein de RDoc, vous ne pouvez
pas utiliser de layouts écrits en RDoc. Toutefois, il est
possible d’utiliser un moteur de rendu différent pour le template et
pour le layout en utilisant l’option `:layout_engine`.

### Templates Radius
<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://radius.rubyforge.org/">radius</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.radius</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>radius :index, :locals => { :key => 'value' }</tt></td>
  </tr>
</table>

Comme vous ne pouvez pas appeler de méthodes Ruby depuis un template
Radius, vous aurez sûrement à lui passer des variables locales.

### Templates Markaby

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://markaby.github.com/">markaby</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.mab</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>markaby { h1 "Bienvenue !" }</tt></td>
  </tr>
</table>

Ce moteur accepte également un bloc pour des templates en ligne (voir
exemple).

### Templates RABL

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="https://github.com/nesquena/rabl">rabl</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.rabl</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>rabl :index</tt></td>
  </tr>
</table>

### Templates Slim

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="http://slim-lang.com/">slim</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.slim</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>slim :index</tt></td>
  </tr>
</table>

### Templates Creole

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="https://github.com/minad/creole">creole</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.creole</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>creole :wiki, :layout_engine => :erb</tt></td>
  </tr>
</table>

Il n'est pas possible d'appeler des méthodes depuis markdown, ni de lui
passer des variables locales. Par conséquent, il sera souvent utilisé en
combinaison avec un autre moteur de rendu :

```ruby
erb :overview, :locals => { :text => markdown(:introduction) }
```

Notez que vous pouvez également appeler la méthode +markdown+ au sein d'autres
templates :

```ruby
%h1 Hello From Haml !
%p= markdown(:greetings)
```

Comme vous ne pouvez pas appeler de Ruby au sein de Markdown, vous ne pouvez
pas utiliser de +layouts+ écrits en Markdown. Toutefois, il est possible
d'utiliser un moteur de rendu différent pour le template et pour le +layout+
en utilisant l'option `:layout_engine`.

### Templates CoffeeScript

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="https://github.com/josh/ruby-coffee-script">coffee-script</a> et un [moyen d'exécuter javascript](https://github.com/sstephenson/execjs/blob/master/README.md#readme)</td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.coffee</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>coffee :index</tt></td>
  </tr>
</table>

### Templates Yajl

<table>
  <tr>
    <td>Dépendances</td>
    <td><a href="https://github.com/brianmario/yajl-ruby">yajl-ruby</a></td>
  </tr>
  <tr>
    <td>Extensions de fichier</td>
    <td><tt>.yajl</tt></td>
  </tr>
  <tr>
    <td>Exemple</td>
    <td><tt>yajl :index, :locals => { :key => 'qux' }, :callback => 'present', :variable => 'resource'</tt></p>
    </td>
  </tr>
</table>

Le source du template est évalué en tant que chaine Ruby, puis la
variable json obtenue est convertie avec #to_json.

```ruby
json = { :foo => 'bar' }
json[:baz] = key
```

Les options `:callback` et `:variable` peuvent être utilisées pour décorer l’objet retourné.

```
var resource = {"foo":"bar","baz":"qux"}; present(resource);</pre>
```

### Templates WLang

<table>
  <tr>
    <td>Dependency</td>
    <td><a href="https://github.com/blambeau/wlang/">wlang</a></td>
  </tr>
  <tr>
    <td>File Extensions</td>
    <td><tt>.wlang</tt></td>
  </tr>
  <tr>
    <td>Example</td>
    <td><tt>wlang :index, :locals => { :key => 'value' }</tt></td>
  </tr>
</table>

L’appel de code ruby au sein des templates n’est pas idiomatique en wlang. L’écriture de templates sans logique est encouragé, via le passage de variables locales. Il est néanmoins possible d’écrire un layout en wlang et d’y utiliser `yield`.

### Templates embarqués

```ruby
get '/' do
  haml '%div.title Bonjour le monde'
end
```

Générera le code du template spécifié dans la chaîne de caractères.

### Accéder aux variables dans un Template

Un template est évalué dans le même contexte que l'endroit d'où il a été
appelé (gestionnaire de route). Les variables d'instance déclarées dans le
gestionnaire de route sont directement accessibles dans le template :

```ruby
get '/:id' do
  @foo = Foo.find(params[:id])
  haml '%h1= @foo.nom'
end
```

Alternativement, on peut passer un hash contenant des variables locales :

```ruby
get '/:id' do
  foo = Foo.find(params[:id])
  haml '%h1= foo.nom', :locals => { :foo => foo }
end
```

Ceci est généralement utilisé lorsque l'on veut utiliser un template comme
partiel (depuis un autre template) et qu'il est donc nécessaire d'adapter les
noms de variables.

### Templates dans le fichier source

Des templates peuvent être définis dans le fichier source comme ceci :

```ruby
require 'sinatra'

get '/' do
  haml :index
end

__END__

@@ layout
%html
  = yield

@@ index
%div.title Bonjour le monde !
```

NOTE : Les templates du fichier source qui contient `require 'sinatra'`
sont automatiquement chargés. Si vous avez des templates dans d'autres
fichiers source, il faut explicitement les déclarer avec
`enable :inline_templates`.

### Templates nommés

Les templates peuvent aussi être définis grâce à la méthode de haut niveau `template` :

```ruby
template :layout do
  "%html\n  =yield\n"
end

template :index do
  '%div.title Bonjour le monde !'
end

get '/' do
  haml :index
end
```

Si un template nommé "layout" existe, il sera utilisé à chaque fois qu'un
template sera affiché. Vous pouvez désactivez les layouts au cas par cas en
passant `:layout => false` ou bien les désactiver par défaut au moyen
de `set :haml, :layout => false` :

```ruby
get '/' do
  haml :index, :layout => !request.xhr?
end
```

### Associer des extensions de fichier

Pour associer une extension de fichier avec un moteur de rendu, utilisez
`Tilt.register`. Par exemple, si vous désirez utiliser l'extension
de fichier `tt` pour les templates Textile, vous pouvez faire comme suit :

```ruby
Tilt.register :tt, Tilt[:textile]
```

### Ajouter son propre moteur de rendu

En premier lieu, déclarez votre moteur de rendu avec Tilt, ensuite créez
votre méthode de rendu :

```ruby
Tilt.register :monmoteur, MonMerveilleurMoteurDeRendu

helpers do
  def monmoteur(*args) render(:monmoteur, *args) end
end

get '/' do
  monmoteur :index
end
```

Utilisera `./views/index.monmoteur`. Voir [le dépôt Github](https://github.com/rtomayko/tilt) pour en savoir plus sur Tilt.

## Filtres

Les filtres before sont exécutés avant chaque requête, dans le même contexte
que les routes, et permettent de modifier la requête et sa réponse. Les
variables d'instance déclarées dans les filtres sont accessibles au niveau
des routes et des templates :

```ruby
before do
  @note = 'Coucou !'
  request.path_info = '/foo/bar/baz'
end

get '/foo/*' do
  @note #=> 'Coucou !'
  params[:splat] #=> 'bar/baz'
end
```

Les filtres after sont exécutés après chaque requête à l'intérieur du même
contexte et permettent de modifier la requête et sa réponse. Les variables
d'instance déclarées dans les filtres before ou les routes sont accessibles
au niveau des filtres after :

```ruby
after do
  puts response.status
end
```

Note : Le corps de la réponse n'est pas disponible au niveau du filtre after
car il ne sera généré que plus tard (sauf dans le cas où vous utilisez la
méthode +body+ au lieu de simplement renvoyer une chaine depuis vos routes).

Les filtres peuvent être associés à un masque, ce qui permet de limiter leur
exécution aux cas où la requête correspond à ce masque :

```ruby
before '/secret/*' do
  authentification!
end

after '/faire/:travail' do |travail|
  session[:dernier_travail] = travail
end
```

Tout comme les routes, les filtres acceptent également des conditions :

```ruby
before :agent => /Songbird/ do
  # ...
end

after '/blog/*', :host_name => 'example.com' do
  # ...
end
```

## Helpers

Utilisez la méthode de haut niveau `helpers` pour définir des routines
qui seront accessibles dans vos gestionnaires de route et dans vos templates :

```ruby
helpers do
  def bar(nom)
    "#{nom}bar"
  end
end

get '/:nom' do
  bar(params[:nom])
end
```

Vous pouvez aussi définir les méthodes helper dans un module séparé :

```ruby
module FooUtils
  def foo(nom) "#{nom}foo" end
end

module BarUtils
  def bar(nom) "#{nom}bar" end
end

helpers FooUtils, BarUtils
```

Cela a le même résultat que d'inclure les modules dans la classe de
l'application.

### Utiliser les sessions

Une session est utilisée pour conserver un état entre les requêtes. Une fois
activées, vous avez un +hash+ de session par session utilisateur :

```ruby
enable :sessions

get '/' do
  "valeur = " << session[:valeur].inspect
end

get '/:value' do
  session[:valeur] = params[:valeur]
end
```

Notez que <tt>enable :sessions</tt> enregistre en fait toutes les données dans
un +cookie+. Ce n'est pas toujours ce que vous voulez (enregistrer beaucoup de
données va augmenter le traffic par exemple). Vous pouvez utiliser n'importe
quel +middleware+ Rack de session afin d'éviter cela. N'utiliser *pas*
<tt>enable :sessions</tt> dans ce cas mais charger le +middleware+ de votre
choix comme vous le feriez pour n'importe quel autre +middleware+ :

```ruby
use Rack::Session::Pool, :expire_after => 2592000

get '/' do
  "valeur = " << session[:valeur].inspect
end

get '/:value' do
  session[:valeur] = params[:valeur]
end
```

Pour renforcer la sécurité, les données de session dans le cookie sont signées
avec une clé secrète de session. Une clé secrète est générée pour vous au
hasard par Sinatra. Toutefois, comme cette clé change à chaque démarrage de
votre application, vous pouvez définir cette clé vous-même afin que toutes
les instances de votre application la partage :

```ruby
set :session_secret, 'super secret'
```

Si vous souhaitez avoir plus de contrôle, vous pouvez également enregistrer un
+hash+ avec des options lors de la configuration de `sessions` :

```ruby
set :sessions, :domain => 'foo.com'
```

### Halt

Pour arrêter immédiatement la requête dans un filtre ou un gestionnaire de
route :

```ruby
halt
```

Vous pouvez aussi passer le code retour ...

```ruby
halt 410
```

Ou le texte ...

```ruby
halt 'Ceci est le texte'
```

Ou les deux ...

```ruby
halt 401, 'Partez !'
```

Ainsi que les entêtes ...

```ruby
halt 402, {'Content-Type' => 'text/plain'}, 'revanche'
```

Bien sûr il est possible de combiner un template avec `halt` :

```ruby
halt erb(:erreur)
```

### Passer

Une route peut passer le relais aux autres routes qui correspondent également
avec `pass` :

```ruby
get '/devine/:qui' do
  pass unless params[:qui] == 'Frank'
  "Tu m'as eu !"
end

get '/devine/*' do
  'Manqué !'
end
```

On sort donc immédiatement de ce gestionnaire et on continue à chercher,
dans les masques suivants, le prochain qui correspond à la requête.
Si aucun des masques suivants ne correspond, un code 404 est retourné.

### Déclencher une autre route

Parfois, +pass+ n'est pas ce que vous recherchez, au lieu de cela vous
souhaitez obtenir le résultat d'une autre route. Pour cela, utilisez
simplement call :

```ruby
get '/foo' do
  status, headers, body = call env.merge("PATH_INFO" => '/bar')
  [status, headers, body.map(&:upcase)]
end

get '/bar' do
  "bar"
end
```

Notez que dans l'exemple ci-dessus, vous faciliterez les tests et améliorerez
la performance en déplaçant simplement `"bar"` dans un helper
utilisé à la fois par `/foo` et `/bar`.

Si vous souhiatez que la requête soit envoyée à la même instance de
l'application plutôt qu'à une copie, utilisez `call!` au lieu de
`call`.

Lisez la spécification Rack si vous souhaitez en savoir plus sur
`call`.

### Définir le corps, le code retour et les entêtes

Il est possible et recommandé de définir le code retour et le corps de la
réponse au moyen de la valeur de retour d'un bloc définissant une route.
Quoiqu'il en soit, dans certains cas vous pourriez avoir besoin de définir
le coprs de la réponse à un moment arbitraire de l'exécution. Vous pouvez le
faire au moyen de la méthode +body+. Si vous faites ainsi, vous pouvez alors
utiliser cette même méthode pour accéder au corps de la réponse :

```ruby
get '/foo' do
  body "bar"
end

after do
  puts body
end
```

Il est également possible de passer un bloc à `body`, qui sera exécuté par le
gestionnaire Rack (ceci peut être utilisé pour implémenter un streaming,
voir "Valeurs de retour").

Pareillement au corps de la réponse, vous pouvez également définir le code
retour et les entêtes :

```ruby
get '/foo' do
  status 418
  headers \
    "Allow"   => "BREW, POST, GET, PROPFIND, WHEN",
    "Refresh" => "Refresh: 20; http://www.ietf.org/rfc/rfc2324.txt"
  body "Je suis une théière !"
end
```

Comme `body` `headers` et `status` peuvent être utilisés sans arguments
pour accéder à leurs valeurs.

### Faire du streaming

Il y a des cas où vous voulez commencer à renvoyer des données pendant que
vous êtes en train de générer le reste de la réponse. Dans les cas les plus
extrèmes, vous souhaitez continuer à envoyer des données tant que le client
n'abandonne pas la connection. Vous pouvez alors utiliser le helper `stream`
pour éviter de créer votre propre système :

```ruby
get '/' do
  stream do |out|
    out << "Ca va être hallu -\n"
    sleep 0.5
    out << " (attends la suite) \n"
    sleep 1
    out << "- cinant !\n"
  end
end
```

Cela permet d'implémenter des API de streaming ou de
[Server Sent Events](http://dev.w3.org/html5/eventsource/) et peut servir de
base pour des [WebSockets](http://en.wikipedia.org/wiki/WebSocket). Vous
pouvez aussi l'employer pour augmenter le débit quand une partie du contenu
provient d'une resource lente.

Le fonctionnement du streaming, notamment le nombre de requêtes simultanées,
dépend énormément du serveur web utilisé. Certains ne prennent pas du tout en
charge le streaming (WEBRick par exemple). Lorsque le serveur ne gère pas le
streaming, la partie body de la réponse sera envoyée au client en une seule
fois, après que l'exécution du bloc passé au helper +stream+ sera terminée. Le
streaming ne fonctionne pas du tout avec Shotgun.

En utilisant le helper +stream+ avec le paramètre +keep_open+, il n'appelera
pas la méthode +close+ du flux, vous laissant la possibilité de le fermer à
tout moment au cours de l'exécution. Ceci ne fonctionne qu'avec les serveurs
evented (ie non threadés) tels que Thin et Rainbows. Les autres serveurs
fermeront malgré tout le flux :

```ruby
set :server, :thin
connections = []

get '/' do
  # conserve le flux ouvert
  stream(:keep_open) { |out| connections << out }
end

post '/' do
  # écrit dans tous les flux ouverts
  connections.each { |out| out << params[:message] << "\n" }
  "message sent"
end
```

### Journalisation (Logging)

Dans le contexte de la requête, la méthode utilitaire +logger+ expose une
instance de +logger+ :

```ruby
get '/' do
  logger.info "chargement des données"
  # ...
end
```

Ce logger va automatiquement prendre en compte les paramètres de
configuration pour la journalisation de votre gestionnaire Rack. Si la
journalisation est désactivée, cette méthode renverra un objet factice et
vous n'avez pas à vous en inquiéter dans vos routes en le filtrant.

Notez que la journalisation est seulement activée par défaut pour
`Sinatra::Application`, donc si vous héritez de `>Sinatra::Base`,
vous aurez à l'activer vous-même :

```ruby
class MonApp < Sinatra::Base
  configure :production, :development do
    enable :logging
  end
end
```

Si vous souhaitez utiliser votre propre logger, vous devez définir le paramètre
`logging` à `nil` pour être certain qu'aucun middleware de logging ne sera
installé (notez toutefois que +logger+ renverra alors +nil+). Dans ce cas,
Sinatra utilisera ce qui sera présent dans `env['rack.logger']`.

### Types Mime

Quand vous utilisez `send_file` ou des fichiers statiques, vous
pouvez rencontrer des types mime que Sinatra ne connaît pas. Utilisez
`mime_type` pour les déclarer par extension de fichier :

```ruby
configure do
  mime_type :foo, 'text/foo'
end
```

Vous pouvez également les utiliser avec la méthode `content_type` :

```ruby
get '/' do
  content_type :foo
  "foo foo foo"
end
```

### Former des URLs

Pour former des URLs, vous devriez utiliser la méthode +url+, par exemple en
Haml :

```ruby
%a{:href => url('/foo')} foo
```

Cela prend en compte les proxy inverse et les routeurs Rack, s'ils existent.

Cette méthode est également disponible sous l'alias +to+ (voir ci-dessous
pour un exemple).

### Redirection du navigateur

Vous pouvez déclencher une redirection du navigateur avec la méthode
`redirect` :

```ruby
get '/foo' do
  redirect to('/bar')
end
```

Tout paramètre additionnel est géré comme des arguments pour la méthode
`halt` :

```ruby
redirect to('/bar'), 303
redirect 'http://google.com', 'mauvais endroit mon pote'
```

Vous pouvez aussi rediriger vers la page dont l'utilisateur venait au moyen de
`redirect back` :

```ruby
get '/foo' do
  "<a href='/bar'>faire quelque chose</a>"
end

get '/bar' do
  faire_quelque_chose
  redirect back
end
```

Pour passer des arguments à une redirection, ajoutez-les soit à la requête :

```ruby
redirect to('/bar?sum=42')
```

Ou bien utilisez une session :

```ruby
enable :sessions

get '/foo' do
  session[:secret] = 'foo'
  redirect to('/bar')
end

get '/bar' do
  session[:secret]
end
```

### Contrôle du cache

Définir correctement vos entêtes à la base pour un bon cache HTTP.

Vous pouvez facilement définir l'entête Cache-Control de la manière suivante :

```ruby
get '/' do
  cache_control :public
  "met le en cache !"
end
```

Conseil de pro : définir le cache dans un filtre +before+ :

```ruby
before do
  cache_control :public, :must_revalidate, :max_age => 60
end
```

Si vous utilisez la méthode +expires+ pour définir l'entête correspondant,
`Cache-Control` sera alors défini automatiquement :

```ruby
before do
  expires 500, :public, :must_revalidate
end
```

Pour utiliser correctement les caches, vous devriez utiliser +etag+ ou
+last_modified+. Il est recommandé d'utiliser ces méthodes *avant* de faire
d'importantes modifications, car elles vont immédiatement déclencher la réponse
si le client a déjà la version courante dans son cache :

```ruby
get '/article/:id' do
  @article = Article.find params[:id]
  last_modified @article.updated_at
  etag @article.sha1
  erb :article
end
```

Il est également possible d'utiliser un
[weak ETag](http://en.wikipedia.org/wiki/HTTP_ETag#Strong_and_weak_validation) :

```ruby
etag @article.sha1, :weak
```

Ces méthodes ne sont pas chargées de mettre des données en cache, mais elles
fournissent les informations nécessaires pour votre cache. Si vous êtes à la
recherche de solutions rapides pour un reverse-proxy de cache, essayez
[rack-cache](https://github.com/rtomayko/rack-cache) :

```ruby
require "rack/cache"
require "sinatra"

use Rack::Cache

get '/' do
  cache_control :public, :max_age => 36000
  sleep 5
  "hello"
end
```

Utilisez le paramètre `:static_cache_control` pour ajouter l'information
d'en-tête `Cache-Control` (voir plus loin).

D'après la RFC 2616, votre application devrait se comporter différement lorsque
l'en-tête If-Match ou If-None-Match est défini à `*` en tenant compte du
fait que la resource demandée existe déjà ou pas. Sinatra considère que les
requêtes portant sur des resources sûres (tel que get) ou idempotentes (tel que
put) existent déjà et pour les autres resources (par exemple dans le cas
de requêtes post) qu'il s'agit de nouvelles resources. Vous pouvez modifier ce
comportement en passant une option `:new_resource` :

```ruby
get '/create' do
  etag '', :new_resource => true
  Article.create
  erb :new_article
end
```

Si vous souhaitez utilisez un ETag faible, utilisez l'option <tt>:kind</tt> :

```
etag '', :new_resource => true, :kind => :weak
```

### Envoyer des fichiers

Pour envoyer des fichiers, vous pouvez utiliser la méthode
`send_file` :

```ruby
get '/' do
  send_file 'foo.png'
end
```

Quelques options sont également acceptées :

```ruby
send_file 'foo.png', :type => :jpg
```

Les options sont :

<dl>
  <dt>filename</dt>
  <dd>
    le nom du fichier dans la réponse, par défaut le nom du fichier envoyé.
  </dd>

  <dt>last_modified</dt>
  <dd>valeur pour l’entête Last-Modified, par défaut la date de modification
du fichier</dd>

  <dt>type</dt>
  <dd>type de contenu à utiliser, deviné à partir de l’extension de fichier
  si absent</dd>

  <dt>disposition</dt>
  <dd>utilisé pour Content-Disposition, les valuers possibles étant :
  `nil` (par défaut), `:attachment` et `:inline`</dd>

  <dt>length</dt>
  <dd>entête Content-Length, par défaut la taille du fichier</dd>

  <dt>status</dt>
  <dd>code état à renvoyer. Utile quand un fichier statique sert de page
  d’erreur.</dd>
</dl>

Si le gestionnaire Rack le supporte, d'autres moyens que le +streaming+ via le
processus Ruby seront utilisés. Si vous utilisez cette méthode, Sinatra gérera
automatiquement les requêtes de type +range+.

### Accéder à l'objet requête

L'objet correspondant à la requête envoyée peut être récupéré dans le contexte
de la requête (filtres, routes, gestionnaires d'erreur) au moyen de la méthode
+request+ :

```ruby
# application tournant à l'adresse http://exemple.com/exemple
get '/foo' do
  t = %w[text/css text/html application/javascript]
  request.accept              # ['text/html', '*/*']
  request.accept? 'text/xml'  # true
  request.preferred_type(t)   # 'text/html'
  request.body                # corps de la requête envoyée par le client
                              # (voir ci-dessous)
  request.scheme              # "http"
  request.script_name         # "/exemple"
  request.path_info           # "/foo"
  request.port                # 80
  request.request_method      # "GET"
  request.query_string        # ""
  request.content_length      # taille de request.body
  request.media_type          # type de média pour request.body
  request.host                # "exemple.com"
  request.get?                # true (méthodes similaires pour les autres
                              # verbes HTTP)
  request.form_data?          # false
  request["UN_ENTETE"]        # valeur de l'entête UN_ENTETE
  request.referer             # référant du client ou '/'
  request.user_agent          # user agent (utilisé par la condition :agent)
  request.cookies             # tableau contenant les cookies du navigateur
  request.xhr?                # requête AJAX ?
  request.url                 # "http://exemple.com/exemple/foo"
  request.path                # "/exemple/foo"
  request.ip                  # adresse IP du client
  request.secure?             # false
  request.forwarded?          # vrai (si on est derrière un proxy inverse)
  request.env                 # tableau brut de l'environnement fourni par
                              # Rack
end
```

Certaines options, telles que `script_name` ou `path_info`
peuvent également être modifiées :

```ruby
before { request.path_info = "/" }

get "/" do
  "toutes les requêtes arrivent ici"
end
```

`request.body` est un objet IO ou StringIO :

```ruby
post "/api" do
  request.body.rewind  # au cas où il a déjà été lu
  donnees = JSON.parse request.body.read
  "Bonjour #{donnees['nom']} !"
end
```

### Fichiers joints

Vous pouvez utiliser la méthode +attachment+ pour indiquer au navigateur que
la réponse devrait être stockée sur le disque plutôt qu'affichée :


```ruby
get '/' do
  attachment
  "enregistre-le !"
end
```

Vous pouvez également lui passer un nom de fichier :

```ruby
get '/' do
  attachment "info.txt"
  "enregistre-le !"
end
```

### Gérer Date et Time

Sinatra fourni un helper +time_for+ pour convertir une valeur donnée en
objet `Time`. Il peut aussi faire la conversion à partir d'objets +DateTime+,
`Date` ou de classes similaires :

```ruby
get '/' do
  pass if Time.now > time_for('Dec 23, 2012')
  "encore temps"
end
```

Cette méthode est utilisée en interne par +expires+, +last_modified+ et
consorts. Par conséquent, vous pouvez très facilement étendre le
fonctionnement de ces méthodes en surchargeant le helper +time_for+ dans
votre application :

```ruby
helpers do
  def time_for(value)
    case value
    when :yesterday then Time.now - 24*60*60
    when :tomorrow  then Time.now + 24*60*60
    else super
    end
  end
end

get '/' do
  last_modified :yesterday
  expires :tomorrow
  "salut"
end
```

### Chercher les fichiers de templates

La méthode `find_template` est utilisée pour trouver les fichiers de
templates à générer :

```ruby
find_template settings.views, 'foo', Tilt[:haml] do |file|
  puts "pourrait être #{file}"
end
```

Ce n'est pas très utilise. En revanche, il est utile de pouvoir surcharger
cette méthode afin de définir son propre mécanisme de recherche. Par exemple,
vous pouvez utiliser plus d'un répertoire de vues :

```ruby
set :views, ['views', 'templates']

helpers do
  def find_template(views, name, engine, &block)
    Array(views).each { |v| super(v, name, engine, &block) }
  end
end
```

Un autre exemple est d'utiliser des répertoires différents pour des moteurs
de rendu différents :

```ruby
set :views, :sass => 'views/sass', :haml => 'templates', :default => 'views'

helpers do
  def find_template(views, name, engine, &block)
    _, folder = views.detect { |k,v| engine == Tilt[k] }
    folder ||= views[:default]
    super(folder, name, engine, &block)
  end
end
```

Vous pouvez également écrire cela dans une extension et la partager avec
d'autres !

Notez que `find_template` ne vérifie pas que le fichier existe mais
va plutôt exécuter le bloc pour tous les chemins possibles. Cela n'induit pas
un problème de performance dans le sens où `render` va utiliser +break+ dès
qu'un fichier est trouvé. De plus, l'emplacement des templates (et leur
contenu) est mis en cache si vous n'êtes pas en mode développement. Vous
devriez garder cela en tête si vous écrivez une méthode vraiment dingue.

## Configuration

Lancé une seule fois au démarrage de tous les environnements :

```ruby
configure do
  # définir un paramètre
  set :option, 'value'

  # définir plusieurs paramètre
  set :a => 1, :b => 2

  # identique à "set :option, true"
  enable :option

  # identique à "set :option, false""
  disable :option

  # vous pouvez également avoir des paramètres dynamiques avec des blocs
  set(:css_dir) { File.join(views, 'css') }
end
```

Lancé si l'environnement (variable d'environnement RACK_ENV) est défini comme
`:production` :

  configure :production do
    ...
  end

Lancé si l'environnement est `:production` ou
`:test` :

  configure :production, :test do
    ...
  end

Vous pouvez accéder à ces paramètres via `settings` :

```
configure do
  set :foo, 'bar'
end

get '/' do
  settings.foo? # => true
  settings.foo  # => 'bar'
  ...
end
```

### Se protéger des attaques

Sinatra utilise [Rack::Protection](https://github.com/rkh/rack-protection#readme)
pour protéger votre application contre les principales attaques opportunistes.
Vous pouvez très simplement désactiver cette fonctionnalité (ce qui exposera
votre application à beaucoup de vulnerabilités courantes) :

```ruby
disable :protection
```

Pour désactiver seulement un type de protection, vous pouvez définir `protection`
avec un hash d'options :

```ruby
set :protection, :except => :path_traversal
```

Vous pouvez également lui passer un tableau pour désactiver plusieurs types de
protection :

```ruby
set :protection, :except => [:path_traversal, :session_hijacking]
```

### Paramètres disponibles

<dl>
  <dt>absolute_redirects</dt>
  <dd>Si désactivé, Sinatra permettra les redirections relatives. Toutefois,
  Sinatra ne sera plus conforme à la RFC 2616 (HTTP 1.1), qui n’autorise
  que les redirections absolues.</p>

  Activez si votre application tourne derrière un proxy inverse qui n’a
  pas été correctement configuré. Notez que la méthode <tt>url</tt>
  continuera de produire des URLs absolues, sauf si vous lui passez
  <tt>false</tt> comme second argument.</p>

  <p>Désactivé par défaut.</p></dd>

  <dt>add_charsets</dt>
  <dd><p>types mime pour lesquels la méthode <tt>content_type</tt> va
  automatiquement ajouter l’information du <tt>charset</tt>.</p>

  <p>Vous devriez lui ajouter des valeurs plutôt que de l’écraser :</p>

  <pre>settings.add_charsets >> "application/foobar"</pre></dd>

  <dt>app_file</dt>
  <dd><p>chemin pour le fichier de l’application principale, utilisé pour
  détecter la racine du projet, les dossiers public et vues, et les
  templates en ligne.</p></dd>

  <dt>bind</dt>
  <dd>adresse IP sur laquelle se brancher (par défaut : 0.0.0.0). Utiliser
  seulement pour le serveur intégré.</dd>

  <dt>default_encoding</dt>
  <dd>encodage à utiliser si inconnu (par défaut <tt>"utf-8"</tt>)</dd>

  <dt>dump_errors</dt>
  <dd>afficher les erreurs dans le <tt>log</tt>.
  </dd>

  <dt>environment</dt>
  <dd>environnement courant, par défaut <tt>ENV['RACK_ENV']</tt>, ou
  <tt>"development"</tt> si absent.</dd>

  <dt>logging</dt>
  <dd>utiliser le <tt>logger</tt>.</dd>

  <dt>lock</dt>
  <dd><p>Place un <tt>lock</tt> autour de chaque requête, n’exécutant donc
  qu’une seule requête par processus Ruby.</p>

  <p>Activé si votre application n’est pas <tt>thread-safe</tt>. Désactivé
  par défaut.</p></dd>

  <dt>method_override</dt>
  <dd>utilise la magie de <tt>_method</tt> afin de permettre des formulaires
  put/delete dans des navigateurs qui ne le permettent pas.

  </dd>
  <dt>port</dt>
  <dd>port à écouter. Utiliser seulement pour le serveur intégré.</dd>

  <dt>prefixed_redirects</dt>
  <dd>si oui ou non <tt>request.script_name</tt> doit être inséré dans les
  redirections si un chemin non absolu est utilisé. Ainsi, <tt>redirect
  '/foo'</tt> se comportera comme <tt>redirect to('/foo')</tt>. Désactivé
  par défaut.</dd>

  <dt>protection</dt>
  <dd>défini s’il faut activer ou non la protection contre les attaques web.
  Voir la section protection précédente.</dd>

  <dt>public_dir</dt>
  <dd>alias pour <tt>public_folder</tt>. Voir ci-dessous.</dd>

  <dt>public_folder</dt>
  <dd>chemin pour le dossier à partir duquel les fichiers publics sont servis.
  Utilisé seulement si les fichiers statiques doivent être servis (voir le
  paramètre <tt>static</tt>). Si non défini, il découle du paramètre
  <tt>app_file</tt>.</dd>

  <dt>reload_templates</dt>
  <dd>si oui ou non les templates doivent être rechargés entre les requêtes.
  Activé en mode développement.</dd>

  <dt>root</dt>
  <dd>chemin pour le dossier racine du projet. Si non défini, il découle du
  paramètre <tt>app_file</tt>.</dd>

  <dt>raise_errors</dt>
  <dd>soulever les erreurs (ce qui arrêtera l’application). Désactivé par
  défaut sauf lorsque <tt>environment</tt> est défini à
  <tt>"test"</tt>.</dd>

  <dt>run</dt>
  <dd>si activé, Sinatra s’occupera de démarrer le serveur, ne pas activer si
  vous utiliser rackup ou autres.</dd>

  <dt>running</dt>
  <dd>est-ce que le serveur intégré est en marche ? ne changez pas ce
  paramètre !</dd>

  <dt>server</dt>
  <dd>serveur ou liste de serveurs à utiliser pour le serveur intégré. Par
  défaut [‘thin’, ‘mongrel’, ‘webrick’], l’ordre indiquant la
  priorité.</dd>

  <dt>sessions</dt>
  <dd>active le support des sessions basées sur les cookies, en utilisant
  <tt>Rack::Session::Cookie</tt>. Reportez-vous à la section ‘Utiliser les
  sessions’ pour plus d’informations.</dd>

  <dt>show_exceptions</dt>
  <dd>affiche la trace de l’erreur dans le navigateur lorsqu’une exception se
  produit. Désactivé par défaut sauf lorsque <tt>environment</tt> est
  défini à <tt>"development"</tt>.</dd>

  <dt>static</dt>
  <dd>Si oui ou non Sinatra doit s’occuper de servir les fichiers statiques.
  Désactivez si vous utilisez un serveur capable de le gérer lui même. Le
  désactiver augmentera la performance. Activé par défaut pour le style
  classique, désactivé pour le style modulaire.</dd>

  <dt>static_cache_control</dt>
  <dd>A définir quand Sinatra rend des fichiers statiques pour ajouter les
  en-têtes <tt>Cache-Control</tt>. Utilise le helper <tt>cache_control</tt>.
  Désactivé par défaut. Utiliser un array explicite pour définir des
  plusieurs valeurs : <tt>set :static_cache_control, [:public, :max_age =>
  300]</tt></dd>

  <dt>threaded</dt>
  <dd>à définir à <tt>true</tt> pour indiquer à Thin d’utiliser
  <tt>EventMachine.defer</tt> pour traiter la requête.</dd>

  <dt>views</dt>
  <dd>chemin pour le dossier des vues. Si non défini, il découle du paramètre
  <tt>app_file</tt>.</dd>
</dl>

## Environements

Il existe trois environnements prédéfinis : `"development"`,
`"production"` et `"test"`. Les environements peuvent être
sélectionné via la variable d'environnement +RACK_ENV+. Sa valeur par défaut
est `"development"`. Dans ce mode, tous les templates sont rechargés à
chaque requête. Des handlers spécifiques pour `not_found` et
`error` sont installés pour vous permettre d'avoir une pile de trace
dans votre navigateur. En mode `"production"` et `"test"` les
templates sont mis en cache par défaut.

Pour exécuter votre application dans un environnement différent, utilisez
l'option `-e` de Ruby :

```bash
$ ruby mon_application.rb -e [ENVIRONMENT]
```

Vous pouvez utiliser une des méthodes +development?+, +test?+ et +production?+
pour déterminer quel est l'environnement en cours.

## Gérer les erreurs

Les gestionnaires d'erreur s'exécutent dans le même contexte que les routes ou
les filtres, ce qui veut dire que vous avez accès (entre autres) aux bons
vieux `haml`, `erb`, `halt`, etc.

### NotFound

Quand une exception <tt>Sinatra::NotFound</tt> est soulevée, ou que le code
retour est 404, le gestionnaire <tt>not_found</tt> est invoqué :

```ruby
not_found do
  'Pas moyen de trouver ce que vous cherchez'
end
```

### Error

Le gestionnaire +error+ est invoqué à chaque fois qu'une exception est
soulevée dans une route ou un filtre. L'objet exception est accessible via la
variable Rack `sinatra.error` :

```ruby
error do
  'Désolé mais une méchante erreur est survenue - ' + env['sinatra.error'].name
end
```

Erreur sur mesure :

```ruby
error MonErreurSurMesure do
  'Donc il est arrivé ceci...' + env['sinatra.error'].message
end
```

Donc si ceci arrive :

```ruby
get '/' do
  raise MonErreurSurMesure, 'quelque chose de mal'
end
```

Vous obtenez ça :

  Donc il est arrivé ceci... quelque chose de mal

Alternativement, vous pouvez avoir un gestionnaire d'erreur associé à un code
particulier :

```ruby
error 403 do
  'Accès interdit'
end

get '/secret' do
  403
end
```

Ou un intervalle :

```ruby
error 400..510 do
  'Boom'
end
```

Sinatra installe pour vous quelques gestionnaires `not_found` et
`error` génériques lorsque vous êtes en environnement
`development`.

## Les Middlewares Rack

Sinatra tourne avec [Rack](http://rack.rubyforge.org/), une interface standard
et minimale pour les web frameworks Ruby. Un des points forts de Rack est le
support de ce que l'on appelle des "middlewares" -- composant qui vient se
situer entre le serveur et votre application, et dont le but est de
visualiser/manipuler la requête/réponse HTTP, et d'offrir diverses
fonctionnalités classiques.

Sinatra permet de construire facilement des middlewares Rack via la méthode de
haut niveau +use+ :

```ruby
require 'sinatra'
require 'mon_middleware_perso'

use Rack::Lint
use MonMiddlewarePerso

get '/bonjour' do
  'Bonjour le monde'
end
```

La sémantique de +use+ est identique à celle définie dans le DSL de
[Rack::Builder](http://rack.rubyforge.org/doc/classes/Rack/Builder.html)
(le plus souvent utilisé dans un fichier rackup). Par exemple, la méthode
+use+ accepte divers arguments ainsi que des blocs :

```
use Rack::Auth::Basic do |login, password|
  login == 'admin' && password == 'secret'
end
```

Rack est distribué avec une bonne variété de middlewares standards pour les
logs, débuguer, faire du routage URL, de l'authentification, gérer des
sessions. Sinatra utilise beaucoup de ces composants automatiquement via la
configuration, donc pour ceux-ci vous n'aurez pas à utiliser la méthode `use`.

## Tester

Les tests pour Sinatra peuvent être écrit avec n'importe quelle bibliothèque
basée sur Rack. [Rack::Test](http://gitrdoc.com/brynary/rack-test) est
recommandé :

```ruby
require 'mon_application_sinatra'
require 'test/unit'
require 'rack/test'

class MonTest < Test::Unit::TestCase
  include Rack::Test::Methods

  def app
    Sinatra::Application
  end

  def test_ma_racine
    get '/'
    assert_equal 'Bonjour le monde !', last_response.body
  end

  def test_avec_des_parametres
    get '/rencontrer', :name => 'Frank'
    assert_equal 'Salut Frank !', last_response.body
  end

  def test_avec_rack_env
    get '/', {}, 'HTTP_USER_AGENT' => 'Songbird'
    assert_equal "Vous utilisez Songbird !", last_response.body
  end
end
```

## Sinatra::Base - Les Middlewares, Bibliothèques, et Applications Modulaires

Définir votre application au niveau supérieur fonctionne bien dans le cas des
micro-applications mais présente pas mal d'inconvénients pour créer des
composants réutilisables sous forme de middlewares Rack, de Rails metal, de
simples librairies avec un composant serveur ou même d'extensions Sinatra. Le
niveau supérieur suppose une configuration dans le style des micro-applications
(une application d'un seul fichier, des répertoires `./public` et
`./views`, des logs, une page d'erreur, etc...). C'est là que
`Sinatra::Base` prend tout son intérêt :

```ruby
require 'sinatra/base'

class MonApplication < Sinatra::Base
  set :sessions, true
  set :foo, 'bar'

  get '/' do
    'Bonjour le monde !'
  end
end
```

Les méthodes de la classe `Sinatra::Base` sont parfaitement identiques à
celles disponibles via le DSL de haut niveau. Il suffit de deux modifications
pour transformer la plupart des applications de haut niveau en un composant
`Sinatra::Base` :

* Votre fichier doit charger `sinatra/base` au lieu de `sinatra`, sinon toutes
  les méthodes du DSL Sinatra seront importées dans l'espace de nom principal.
* Les gestionnaires de routes, la gestion d'erreur, les filtres et les options
  doivent être placés dans une classe héritant de `Sinatra::Base`.

`Sinatra::Base` est une page blanche. La plupart des options sont
désactivées par défaut, y compris le serveur intégré. Reportez-vous à
[Options et Configuration](http://sinatra.github.com/configuration.html)
pour plus d'informations sur les options et leur fonctionnement.

### Style modulaire vs. style classique

Contrairement aux idées reçues, il n'y a rien de mal à utiliser le style
classique. Si c'est ce qui convient pour votre application, vous n'avez pas
aucune raison de passer à une application modulaire.

Le principal inconvénient du style classique sur le style modulaire est que vous
ne pouvez avoir qu'une application Ruby par processus Ruby. Si vous pensez en
utiliser plus, passez au style modulaire. Et rien ne vous empêche de mixer style
classique et style modulaire.

Si vous passez d'un style à l'autre, souvenez-vous des quelques différences
mineures en ce qui concerne les paramètres par défaut :

  Paramètre         Classique                   Modulaire

  app_file          fichier chargeant sinatra   fichier héritant de Sinatra::Base
  run               $0 == app_file              false
  logging           true                        false
  method_override   true                        false
  inline_templates  true                        false
  static            true                        false

### Servir une application modulaire

Il y a deux façons de faire pour démarrer une application modulaire, démarrez
avec `run!` :

```ruby
# my_app.rb
require 'sinatra/base'

class MyApp < Sinatra::Base
  # ... code de l'application ici ...

  # démarre le serveur si ce fichier est directement exécuté
  run! if app_file == $0
end
```

Démarrez ensuite avec :

```bash
$ ruby my_app.rb
```

Ou alors avec un fichier `config.ru`, qui permet d'utiliser n'importe
quel gestionnaire Rack :

```ruby
# config.ru
require './my_app'
run MyApp
```

Exécutez :

```bash
$ rackup -p 4567
```

### Utiliser une application de style classique avec un fichier config.ru

Ecrivez votre application :

```ruby
# app.rb
require 'sinatra'

get '/' do
  'Bonjour le monde !'
end
```

Et un fichier `config.ru` correspondant :

```ruby
require './app'
run Sinatra::Application
```

### Quand utiliser un fichier config.ru ?

Quelques cas où vous devriez utiliser un fichier `config.ru` :

* Vous souhaitez déployer avec un autre gestionnaire Rack (Passenger, Unicorn,
  Heroku, ...).
* Vous souhaitez utiliser plus d'une sous-classe de `Sinatra::Base`.
* Vous voulez utiliser Sinatra comme un middleware, non en tant que
  endpoint.

**Il n'est pas nécessaire de passer par un fichier `config.ru` pour la
seule raison que vous êtes passé au style modulaire, et vous n'avez pas besoin
de passer au style modulaire pour utiliser un fichier `config.ru`.**

### Utiliser Sinatra comme Middleware

Non seulement Sinatra peut utiliser d'autres middlewares Rack, il peut
également être à son tour utilisé au-dessus de n'importe quel endpoint Rack
en tant que middleware. Ce endpoint peut très bien être une autre
application Sinatra, ou n'importe quelle application basée sur Rack
(Rails/Ramaze/Camping/...) :

```ruby
require 'sinatra/base'

class EcranDeConnexion < Sinatra::Base
  enable :sessions

  get('/connexion') { haml :connexion }

  post('/connexion') do
    if params[:nom] = 'admin' && params[:motdepasse] = 'admin'
      session['nom_utilisateur'] = params[:nom]
    else
      redirect '/connexion'
    end
  end
end

class MonApp < Sinatra::Base
  # le middleware sera appelé avant les filtres
  use EcranDeConnexion

  before do
    unless session['nom_utilisateur']
      halt "Accès refusé, merci de vous <a href='/connexion'>connecter</a>."
    end
  end

  get('/') { "Bonjour #{session['nom_utilisateur']}." }
end
```

### Création dynamique d'applications

Il se peut que vous ayez besoin de créer une nouvelle application à l'exécution
sans avoir à les assigner à une constante, vous pouvez le faire grâce à
`Sinatra.new` :

```ruby
require 'sinatra/base'
mon_app = Sinatra.new { get('/') { "salut" } }
mon_app.run!
```

L'application dont elle hérite peut être passé en argument optionnel :

```ruby
# config.ru
require 'sinatra/base'

controleur = Sinatra.new do
  enable :logging
  helpers MyHelpers
end

map('/a') do
  run Sinatra.new(controleur) { get('/') { 'a' } }
end

map('/b') do
  run Sinatra.new(controleur) { get('/') { 'b' } }
end
```

C'est notamment utile pour tester des extensions à Sinatra ou bien pour
utiliser Sinatra dans votre propre bibliothèque.

Cela permet également d'utiliser très facilement Sinatra comme middleware :

```ruby
require 'sinatra/base'

use Sinatra do
  get('/') { ... }
end

run RailsProject::Application
```

## Contextes et Binding

Le contexte dans lequel vous êtes détermine les méthodes et variables
disponibles.

### Contexte de l'application/classe

Toute application Sinatra correspond à une sous-classe de `Sinatra::Base`.
Si vous utilisez le DSL haut niveau (`require 'sinatra'`), alors cette
classe est `Sinatra::Application`, sinon il s'agit de la sous-classe que
vous avez définie. Dans le contexte de la classe, vous avez accès aux méthodes
telles que `get` ou `before`, mais vous n'avez pas accès aux objets +request+
ou +session+ car c'est la même classe d'application qui traitera toutes les
requêtes.

Les options définies au moyen de +set+ deviennent des méthodes de classe :

```ruby
class MonApp < Sinatra::Base
  # Eh, je suis dans le contexte de l'application !
  set :foo, 42
  foo # => 42

  get '/foo' do
    # Eh, je ne suis plus dans le contexte de l'application !
  end
end
```

Vous avez le binding du contexte de l'application dans :

* Le corps de la classe d'application
* Les méthodes définies par les extensions
* Le bloc passé à `helpers`
* Les procs/blocs utilisés comme argument pour `set`
* Le bloc passé à `Sinatra.new`

Vous pouvez atteindre ce contexte (donc la classe) de la façon suivante :

* Via l'objet passé dans les blocs `configure` (`configure { |c| ... }`)
* En utilisant `settings` dans le contexte de la requête

### Contexte de la requête/instance

Pour tout traitement d'une requête, une nouvelle instance de votre classe
d'application est créée et tous vos gestionnaires sont exécutés dans ce
contexte. Dans ce dernier, vous pouvez accéder aux objets `request` et
`session` et faire appel aux fonctions de rendu telles que `erb` ou `haml`.
Vous pouvez accéder au contexte de l'application depuis le contexte de la
requête au moyen de `settings` :

```ruby
class MonApp < Sinatra::Base
  # Eh, je suis dans le contexte de l'application !
  get '/ajouter_route/:nom' do
    # Contexte de la requête pour '/ajouter_route/:nom'
    @value = 42

    settings.get("/#{params[:nom]}") do
      # Contexte de la requête pour "/#{params[:nom]}"
      @value # => nil (on est pas au sein de la même requête)
    end

    "Route ajoutée !"
  end
end
```

Vous avez le binding du contexte de la requête dans :

* les blocs get/head/post/put/delete/options
* les filtres before/after
* les méthodes utilitaires (définies au moyen de `helpers`)
* les vues/templates

### Le contexte de délégation

Le contexte de délégation se contente de transmettre les appels de méthodes au
contexte de classe. Toutefois, il ne se comporte pas à 100% comme le contexte
de classe car vous n'avez pas le binding de la classe : seules les méthodes
spécifiquement déclarées pour délégation sont disponibles et il n'est pas
possible de partager des variables/états avec le contexte de classe
(comprenez : `self` n'est pas le même). Vous pouvez ajouter des délégation de
méthodes en appelant `Sinatra::Delegator.delegate :method_name`.

Vous avez le binding du contexte de délégation dans :

* Le binding de haut niveau, si vous avez utilisé `require "sinatra"`
* Un objet qui inclut le module `Sinatra::Delegator`

Jetez un oeil pour vous faire une idée : voici le
[mixin Sinatra::Delegator](https://github.com/sinatra/sinatra/blob/ca06364/lib/sinatra/base.rb#L1609-1633)
qui [étend l'objet principal](https://github.com/sinatra/sinatra/blob/ca06364/lib/sinatra/main.rb#L28-30).

## Ligne de commande

Les applications Sinatra peuvent être lancées directement :

```ruby
$ ruby mon_application.rb [-h] [-x] [-e ENVIRONNEMENT] [-p PORT] [-o HOTE] [-s SERVEUR]
```

Les options sont :

```
-h # aide
-p # déclare le port (4567 par défaut)
-o # déclare l'hôte (0.0.0.0 par défaut)
-e # déclare l'environnement (+development+ par défaut)
-s # déclare le serveur/gestionnaire à utiliser (thin par défaut)
-x # active le mutex lock (off par défaut)
```

## Configuration nécessaire

Les versions suivantes de Ruby sont officiellement supportées :

<dl>
  <dt>Ruby 1.8.7</dt>
  <dd>1.8.7 est complètement supporté, toutefois si rien ne vous en empêche,
  nous vous recommandons de passer à 1.9.2 ou bien de passer à JRuby ou
  Rubinius. Le support de Ruby 1.8.7 ne sera pas supprimé avant la sortie de
  Sinatra 2.0 et de Ruby 2.0, à moins qu’un improbable Ruby 1.8.8
  apparaisse. Et même dans ce cas, nous pourrions continuer à le supporter.
  **Ruby 1.8.6 n’est plus supporté**. Si vous souhaitez utiliser la
  version 1.8.6, vous devez revenir à Sinatra 1.2 qui continuera à recevoir
  des corrections de bugs tant que Sinatra 1.4.0 ne sera pas livré.</dd>

  <dt>Ruby 1.9.2</dt>
  <dd>1.9.2 est totalement supporté et recommandé. N’utilisez pas 1.9.2p0 car
  il provoque des erreurs de segmentation à l’exécution de Sinatra. Son
  support continuera au minimum jusqu’à la sortie de Ruby 1.9.4/2.0 et le
  support de la dernière version 1.9 se poursuivra aussi longtemps que la
  core team de Ruby la supportera.</dd>

  <dt>Ruby 1.9.3</dt>
  <dd>1.9.3 est totalement supporté et recommandé. Nous vous rappelons que
  passer à 1.9.3 depuis une version précédente annulera toutes les
  sessions.</dd>


  <dt>Rubinius</dt>
  <dd>Rubinius est officiellement supporté (Rubinius <= 1.2.4), tout
  fonctionne, y compris tous les langages de template. La version 2.0 à
  venir est également supportée.</dd>

  <dt>JRuby</dt>
  <dd>JRuby est officiellement supporté (JRuby <= 1.6.5). Aucune anomalie
  avec des bibliothèques de templates tierces ne sont connues. Toutefois, si
  vous choisissez JRuby, alors tournez vous vers des gestionnaires Rack JRuby
  car le serveur Thin n’est pas complètement supporté par JRuby. Le
  support des extensions C dans JRuby est encore expérimental, ce qui
  n’affecte que RDiscount, Redcarpet and RedCloth pour l’instant.</dd>
</dl>

Nous gardons également un oeil sur les versions Ruby à venir.

Les implémentations Ruby suivantes ne sont pas officiellement supportées mais
sont malgré tout connues pour permettre de faire fonctionner Sinatra :

* Versions plus anciennes de JRuby et Rubinius
* Ruby Enterprise Edition
* MacRuby, Maglev, IronRuby
* Ruby 1.9.0 et 1.9.1 (mais nous déconseillons leur utilisation)

Le fait de ne pas être officiellement supporté signifie que si quelque chose
ne fonctionne pas uniquement sur cette plateforme alors c'est un problème de la
plateforme et pas un bug de Sinatra.

Nous lançons également notre intégration continue (CI) avec ruby-head (la
future 2.0.0) et la branche 1.9.4, mais étant donné les évolutions continuelles,
nous ne pouvont rien garantir, si ce n'est que les versions 1.9.4p0 et 2.0.0p0
seront supportées.

Sinatra devrait fonctionner sur n'importe quel système d'exploitation
supportant l'implémentation Ruby choisie.

Il n'est pas possible d'utiliser Sinatra sur Cardinal, SmallRuby, Blueuby ou
toute version de Ruby antérieure à 1.8.7 à l'heure actuelle.

## Essuyer les plâtres

Si vous voulez utiliser la toute dernière version de Sinatra, n'ayez pas peur
de faire tourner votre application sur la branche master, cela devrait être
stable.

Nous publions également une gem de +prerelease+ de temps en temps que vous
pouvez installer comme suit :

```ruby
$ gem install sinatra --pre
```

afin d'avoir les toutes dernières fonctionnalités.

### Avec Bundler

Si vous voulez faire tourner votre application avec le tout dernier
Sinatra, [Bundler](http://gembundler.com/) est recommandé.

Tout d'abord, installer bundler si vous ne l'avez pas :

```bash
$ gem install bundler
```

Ensuite, dans le dossier de votre projet, créez un fichier +Gemfile+ :

```ruby
source :rubygems
gem 'sinatra', :git => "git://github.com/sinatra/sinatra.git"

# autres dépendances
gem 'haml'                    # par exemple, si vous utilisez haml
gem 'activerecord', '~> 3.0'  # peut-être que vous avez également besoin
                              # de ActiveRecord 3.x
```

Notez que vous aurez à lister toutes les dépendances de votre application dans
ce fichier. Les dépendances directes de Sinatra (Rack et Tilt) seront
automatiquement téléchargées et ajoutées par Bundler.

Maintenant, vous pouvez faire tourner votre application de la façon suivante :

```bash
$ bundle exec ruby myapp.rb
```

### Faites le vous-même

Créez un clone local et démarrez votre application avec le dossier
`sinatra/lib` dans le `$LOAD_PATH` :

```bash
$ cd myapp
$ git clone git://github.com/sinatra/sinatra.git
$ ruby -Isinatra/lib myapp.rb

A l'avenir, pour mettre à jour le code source de Sinatra :

```bash
$ cd myapp/sinatra
$ git pull
```

### Installez globalement

Vous pouvez construire la gem vous-même :

```bash
$ git clone git://github.com/sinatra/sinatra.git
$ cd sinatra
$ rake sinatra.gemspec
$ rake install
```

Si vous installez les gems en tant que +root+, la dernière étape sera :

```bash
$ sudo rake install
```

## Versions

Sinatra se conforme aux (versions sémantiques)[http://semver.org/], aussi bien
SemVer que SemVerTag.

## Mais encore

* [Site internet](http://www.sinatrarb.com/) - Plus de documentation,
  de news, et des liens vers d'autres ressources.
* [Contribuer](http://www.sinatrarb.com/contributing) - Vous avez trouvé un
  bug ? Besoin d'aide ? Vous avez un patch ?
* [Suivi des problèmes](http://github.com/sinatra/sinatra/issues)
* [Twitter](http://twitter.com/sinatra)
* [Mailing List])(http://groups.google.com/group/sinatrarb/topics)
* [IRC : #sinatra](irc://chat.freenode.net/#sinatra) sur http://freenode.net
* [IRC : #sinatra](irc://chat.freenode.net/#sinatra) on http://freenode.net
* [Sinatra Book](http://sinatra-book.gittr.com) Tutoriels et recettes
* [Sinatra Recipes](http://recipes.sinatrarb.com/) trucs et astuces rédigés par 
  la communauté
* Documentation API de la [dernière version](http://rubydoc.info/gems/sinatra)
  ou du [HEAD courant](http://rubydoc.info/github/sinatra/sinatra) sur
  http://rubydoc.info
* [CI server](http://travis-ci.org/sinatra/sinatra)