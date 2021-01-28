# sumfony-guide

> dependence to install : 
- php namespace resolver
- twig 
- easy html
- easy css
- symfony for vscode

> 1/ Make entity named User with property `email`,`username` and `password` : 
```php
php bin/console make:entity User
```
> continue with migration ! 
```php
php bin/console make:migration
php bin/console doctrine:migrations:migrate
```
> 2/ make a registration form for the `Entity` __User__ : 
```php
php bin/console make:form RegistrationType

 The name of Entity or fully qualified model class name that the new form will be bound to (empty for none):
 > User
 ```
 
> 3/ Validation of password field ! 
```php
 /**
     * @ORM\Column(type="string", length=255)
     * @Assert\Length(min="8", minMessage="votre mot de passe doit etre au min 8 caracteres ")
     * @Assert\EqualTo(propertyPath="confirm_password", message="vous n'aves pas tapé le meme mot de passe")
     */
    private $password;

    /**
     * @Assert\EqualTo(propertyPath="password",message="vous n'aves pas tapé le meme mot de passe")
     */
    public $confirm_password;
```
> Exemple of securityController :
```php
<?php

namespace App\Controller;

use App\Entity\User;
use App\Form\RegistrationType;
use Doctrine\Persistence\ObjectManager;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;

class SecurityController extends AbstractController
{
   
    /**
     * @Route("/inscription", name="security_registration")
     */
    public function registration(Request $request) {
        
        $user = new User();
        $entityManager = $this->getDoctrine()->getManager();
        $form = $this->createForm(RegistrationType::class, $user);

        $form->handleRequest($request);

        if($form->isSubmitted() && $form->isValid()) {
            $entityManager->persist($user);
            $entityManager->flush();
        }

        return $this->render('security/registration.html.twig', [
            'form' => $form->createView()
        ]);
    }
}
```
> hide __password__ when typing `Registration Form`
```php
class RegistrationType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder
            ->add('email')
            ->add('username')
            ->add('password', PasswordType::class)
            ->add('confirm_password', PasswordType::class)
        ;
    }
```
<hr>

> Registraion.html.twig : 
```php
{% extends 'base.html.twig' %}

{% block body %}
<h1>Inscription sur le site </h1>

        {{ form_start(form) }}

        {{ form_row(form.username, {'label': 'Nom d\'utilisateur','attr': {'placeholder': "Nom d\'utilisateur' ..."}} ) }}
        {{ form_row(form.email, {'label': 'Adresse email','attr': {'placeholder': "Adresse email..."}} ) }}
        {{ form_row(form.password, {'label': 'Mot de passe','attr': {'placeholder': "Mot de passe..."}} ) }}
         {{ form_row(form.confirm_password, {'label': 'Répetez votre Mot de passe','attr': {'placeholder': "répetez votre Mot de passe..."}} ) }}

        {{ form_widget(form) }}

        <button type="submit" class="btn btn-success">Inscription</button>
        {{ form_end(form) }}


{% endblock %}
```
<hr>

> 3/ crypt the password woth `bcrypt` : 
- add this syntax to security.yaml : 
```php
security:
    encoders:
        App\Entity\User:
            algorithm: bcrypt
```
- Make the password `encrypted` before __persisting__ from the registration form : 
```php
 /**
     * @Route("/inscription", name="security_registration")
     */
    public function registration(Request $request, UserPasswordEncoderInterface $encoder) {

        $entityManager = $this->getDoctrine()->getManager();
        $user = new User();
        
        $form = $this->createForm(RegistrationType::class, $user);

        $form->handleRequest($request);

        if($form->isSubmitted() && $form->isValid()) {

          
            $hash = $encoder->encodePassword($user, $user->getPassword());

            $user->setPassword($hash);
            
            $entityManager->persist($user);
            $entityManager->flush();
        }

        return $this->render('security/registration.html.twig', [
            'form' => $form->createView()
        ]);
    }
```
> Make login Form with validation : 
- Create 2 root for `connexion` et `déconnecion` :
```php
/**
     * @Route("/connexion", name="security_login")
     */
    public function login() {
        return $this->render('security/login.html.twig');
    }

    /**
     * @Route("/deconnexion", name="security_logout")
     */
    public function logout() {}
```    

- Add some config to `security.yaml` : 
```php
security:
    encoders:
        App\Entity\User:
            algorithm: bcrypt

    # https://symfony.com/doc/current/security.html#where-do-users-come-from-user-providers
    providers:
        users_in_memory: { memory: null }
        in_database:
            entity:
                class: App\Entity\User
                property: email
    firewalls:
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false
        main:
            anonymous: true
            lazy: true
            provider: in_database
            
            form_login:
                login_path: security_login
                check_path: security_login

            logout:
                path: security_logout
                target: blog
                
                //
                   access_control:
        # - { path: ^/admin, roles: ROLE_ADMIN }
        # - { path: ^/profile, roles: ROLE_USER }
```                
