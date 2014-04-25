###Elasticsearch rails 


Dodati sledece u Gemfile:

    gem 'elasticsearch-model', git: 'git://github.com/elasticsearch/elasticsearch-rails.git'
    gem 'elasticsearch-rails', git: 'git://github.com/elasticsearch/elasticsearch-rails.git'


U Modelu tj. Model.rb fajlu dodati:

    require 'elasticsearch/model'

    class Article < ActiveRecord::Base
      include Elasticsearch::Model
    end

Bice podesen [client](https://github.com/elasticsearch/elasticsearch-ruby/tree/master/elasticsearch) koji je povezan sa `localhost:9200` , po default-u. Za dalja podesavanja  
vezana za klijent posjetite navedeni link.

Prebacivanje podataka u elasticsearch server je moguce sledecom komandom:

    Article.import
    # => 0

Drugi nacin na kojim je moguce importovati podatke na elasticsearch je pomocu rake task-a:

Kreirajte fajl `lib/tasks/elasticsearch.rake` i u njemu dodamo sljedecu liniju:

    require 'elasticsearch/rails/tasks/import'

Da bi importovali zapise iz base koristi se sljedeca komanda: 

    bundle exec rake environment elasticsearch:import:model CLASS='Article'

(umjesto Article staviti ime vaseg modela)

Za dodatne instrukcije i opcije vezane pokrenite sljedecu komandu:

    bundle exec rake -D elasticsearch


Slijede osnovne komande za pretragu kao i pozivi vezani za rezultat pretrage.

    response = Article.search 'fox dogs'

    response.results.total
    # => 2

    response.results.first._score
    # => 0.02250402

    response.results.first._source.title
    # => "Quick brown fox"

Radimo sa response objektom koji predstavlja 'wrapper' oko JSON-a koji je vracen kao rezultat  
iz Elasticsearch-a i koji omogucava pristup podacima rezultata ('hits').

Svaki 'hit' je ugnjezden u Result klasu i omogucava pristup 'properties-ima' preko Hashie::Mash

Na `results` objektu je omogucena primjena `Enumerable` interfejsa: 

    response.results.map { |r| r._source.title }
    # => ["Quick brown fox", "Fast black dogs"]

    response.results.select { |r| r.title =~ /^Q/ }
    # => [#<Elasticsearch::Model::Response::Result:0x007 ... "_source"=>{"title"=>"Quick brown fox"}}>]

Pored mogucnosti za dohvatanje rezultata iz Elasticsearch-a mozemo da vratimo kolekciju instanci modela iz nase baze metodom `records`

    response.records.to_a
    # Article Load (0.3ms)  SELECT "articles".* FROM "articles" WHERE "articles"."id" IN (1, 2)
    # => [#<Article id: 1, title: "Quick brown fox">, #<Article id: 2, title: "Fast black dogs">]

    response.records.where(title: 'Quick brown fox').to_a
    # Article Load (0.2ms)  SELECT "articles".* FROM "articles" WHERE "articles"."id" IN (1, 2) AND "articles"."title" = 'Quick brown fox'
    # => [#<Article id: 1, title: "Quick brown fox">]

    response.records.records.class
    # => ActiveRecord::Relation::ActiveRecord_Relation_Article

U vecini slucajeva imacemo potrebu za definisanje `search`-a u sklad sa potrebama, i to se moze realizovati upotrebom Elasticsearch query DSL-a.

Dodatne opcije kao i podesavanja vezana za indexe, callback-ove mozete naci u dokumentaciji  
za ovaj gem na [linku](https://github.com/elasticsearch/elasticsearch-rails/blob/master/elasticsearch-model/README.md)

Sto se tice serializacije modela koristi se po default-u `as_indexed_json` funkcija koja  
definisana u `Elasticsearch::Model::Serializing` modulu i moguce je override-ovati i  
prilagoditi sopstvenim potrebama.

    class Article
      include Elasticsearch::Model

      def as_indexed_json(options={})
        as_json(only: 'title')
      end
    end

    Article.first.as_indexed_json
    # => {"title"=>"Quick brown fox"}

Kada imamo slozeniju strukturu/shemu neophodno je kreirati sopstvenu as_indexed_json funkciju, za primjer je uzeta situacija gdje Article model ima vise Comment-a, Author-a itd.

    def as_indexed_json(options={})
      self.as_json(
        include: { categories: { only: :title},
                   authors:    { methods: [:full_name], only: [:full_name] },
                   comments:   { only: :text }
                 })
    end

    Article.first.as_indexed_json
    # => { "id"         => 1,
    #      "title"      => "First Article",
    #      "created_at" => 2013-12-03 13:39:02 UTC,
    #      "updated_at" => 2013-12-03 13:39:02 UTC,
    #      "categories" => [ { "title" => "One" } ],
    #      "authors"    => [ { "full_name" => "John Smith" } ],
    #      "comments"   => [ { "text" => "First comment" } ] }