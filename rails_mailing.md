#1. Action Mailer

Action mailer omogucava slanje mail-ova iz nase aplikacije.

###1.1 Osnova podesavanja i funkcije

Prvo se mora kreirati mailer, to je moguce pomocu generatora  ili rucno kao npr:

    $ rails generate mailer UserMailer
    create  app/mailers/user_mailer.rb
    invoke  erb
    create    app/views/user_mailer
    invoke  test_unit
    create    test/mailers/user_mailer_test.rb

Maileri su u sustini slicni kontrolerima, isto imaju metode 'akcije' i koriste view-ove  
da strukturiraju sadrzaj. Gdje kontroler generise sadrzaj kao HTML da posalje nazad klijentu 
, mailer kreira poruku koja ce biti poslata kao email.

    class MyMailer < ActionMailer::Base
      default from: 'notifications@example.com'
 
		  def welcome_email(user)
		    @user = user
		    mail(to: @user.email, subject: 'Welcome to My Awesome Site')
		  end
    end

Ovo je primjer nekog osnovnog izgleda mailera.

+ default hash : definishe adresu sa koje se salje (:from heder se prosledjuje)
+ mail : glavna fukncija za slanje (prosledjuju se :to i : subject hederi)

Kao i kod kontrolera sve instance variable koje definisemo su dostupne u view-ovima.

###1.2 Kreiranje View-a

Na sljedecem primjeru je prikazan mailer view pod imenom `welcome_email.html.erb` u  
`app/views/user_mailer/`

    <!DOCTYPE html>
    <html>
      <head>
        <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
      </head>
      <body>
        <h1>Welcome to example.com, <%= @user.name %></h1>
        <p>
          You have successfully signed up to example.com,
          your username is: <%= @user.login %>.<br>
        </p>
        <p>
          To login to the site, just follow this link: <%= @url %>.
        </p>
        <p>Thanks for joining and have a great day!</p>
      </body>
    </html>

Obratiti paznju na imenovanje view-a. View-ovi se nalaze u folderu koji se zove isto kao i  
mailer klasa. Konkretni mailer view je poznat klasi jer mu je ime isto kao mailer funkciji  
(u ovom slucaju welcome_email).

Smatra se za best practice da se pored html poruke salje i text poruka npr: 

    Welcome to example.com, <%= @user.name %>
    ===============================================
 
    You have successfully signed up to example.com,
    your username is: <%= @user.login %>.
 
    To login to the site, just follow this link: <%= @url %>.
 
    Thanks for joining and have a great day!

Kada se pozove mail metoda , ActionMailer prepoznaje dva templejta (text i HTML) i  
automatski generise multipart/alternative email.

Pretpostavimo da imamo User model, dodajemo u odgovarajucem UsersController-u poziv  
mail funkcije za dostavu email-a novokreiranom korisniku kao npr:

    class UsersController < ApplicationController
      # POST /users
      # POST /users.json
      def create
        @user = User.new(params[:user])

        if @user.save
          # Tell the UserMailer to send a welcome email after save
          UserMailer.welcome_email(@user).deliver
        end
      end
    end

Metoda welcome_email vraca Mail::Message objekat, na njemu se poziva `deliver` funckija za  
konacno slanje mail-a.

Za dodatne opcije kao sto su dodavanje attachmenta, layout-i slanje vecem broju korisnika i sl. posjetiti zvanicni [Guide](http://guides.rubyonrails.org/action_mailer_basics.html).

###1.3 Podesavanje za rad sa devise-om

Za rad ActionMailer-a sa devise-om potrebno je kreirati registrations kontroler koji ce nasledjivati devise-ov registrations kontroler i override-ovati
njegovu create metodu kao na sljedecem primjeru:

    class RegistrationsController < Devise::RegistrationsController

	    def create
		    super
		    UserMailer.welcome_email(@user).deliver
	    end
	
    end

#2 Podesavanje mandrill servisa

U development.rb (primjeri su svi na develompment environment-u) dodati sljedece: 

    config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }

    config.action_mailer.delivery_method = :mandrill_delivery

      config.action_mailer.smtp_settings = {
        address:              'smtp.mandrillapp.com',
        port:                 587,
        domain:               'gmail.com',
        user_name:            'your-mail@gmail.com',
        password:             Mandrill-API-KEY,
        authentication:       'plain',
        enable_starttls_auto: true  }

Napomena(ako nemate api-kljuc): Mandrill-API-KEY dobijamo registracijom na [mandrill](https://mandrillapp.com) i aktivacijom api kljuca.
Koristicemo primjer obicnog mailer-a sa funckiom welcome_email:

    class UserMailer < ActionMailer::Base
      default from: "your-mail@gmail.com"

      def welcome_email(user)
  	    @user = user
  	    mail(to: @user.email, subject: 'Thanks for logging in !')		  	
      end
  
    end

Koristicemo za primjer oba view-a iz prvog poglavlja .

U app/models kreiramo fajl `mandrill_delivery.rb` i u njemu: 

    require 'mandrill'

    class MandrillDelivery
      attr_accessor :message

      def initialize(mail)
      end

      def deliver!(mail)
        build_meta_mandrill(mail)
        send_mandrill(mail)
      end

      def build_meta_mandrill(mail)
        #build Mandrill message hash
        @message = {
          :from_name=> "Admin",
          :from_email=>"your-mail@gmail.com",
          :subject=> "#{mail['subject']}",
          :to=>[
                {
                  :email=> "#{mail['to']}"
                }
               ],
          :auto_text => true,
          :global_merge_vars => [
                                 {
                                   :name => "LISTCOMPANY",
                                   :content => "Company Name Here"
                                 }
                                ]
        }

        true
      end

      #sends email via Mandrill
      def send_mandrill(mail)
        m = Mandrill::API.new Mandrill-API-KEY

        sending = m.messages.send(@message)
        Rails.logger.info sending
      end
    end

U /config/initializers kreiramo fajl `mailer.rb` :

    ActionMailer::Base.add_delivery_method :mandrill_delivery, MandrillDelivery

Sami poziv je realizovan kao u 1.3 poglavlju.


#3 MailChimp email lista, integracija sa rails aplikacijom

Na samom pocetku se moramo registrovati na [mailchimp](http://mailchimp.com/).
Kreiramo novu listu. Potreban nam je API kljuc koji dobijamo iz *Extras* dropdown-a account menija i list_id koji nam je dostupan na 
settings dropdown-u (prva stavka) u lists meniju glavnog dashboard-a.

Dodati u gemfile:

    gem 'gibbon', github: 'amro/gibbon'


Zatim u config/initializers dodamo fajl `mailchimp.rb` sadrzaja:

    if Rails.env.test?
      Gibbon::Export.api_key = "fake" 
      Gibbon::Export.throws_exceptions = false 
  	end
    Rails.configuration.mailchimp = Gibbon::API.new(Mailchimp-API-KEY)

Kreiramo na User modelu novu funkciju `subscribe_to_mailchimp` :

    def subscribe_to_mailchimp testing=false
       return true if (Rails.env.test? && !testing)

       list_id = Mailchimp-List_id

       response = Rails.configuration.mailchimp.lists.subscribe({
        id: list_id,
        email: {email: email},
        double_optin: false
        })
       response
    end

Ova funkcija odma vraca true u test modu i time sprecava pozivanje mailchimp live API-ja.
Zatim se dohvata list_id i poziva **lists.subscribe** na mailchimp.

 Postavljeno je double_optin na false, sto sprecava da mailchimp salje korisnicima dodatne mail-ove potvrde jer korisnika dodajemo u listu samim tim sto se registrovao na nasu aplikaciju i nema potrebe za daljim pitanjima.

I na kraju dodajemo after_create callback:

     after_create do
       subscribe_to_mailchimp 
     end