---
layout: post
title: Catching nice exceptions from Moose
---

[Moose](https://metacpan.org/module/Moose) is great. Before Moose creating objects in Perl was a hassle; 
you had to write your own constructor, methods for accessors and 
mutators and at the same time try to provide a clean interface and 
ensure encapsulation. It was tedious. With Moose you can build your 
classes without having to include lots of cruft, leaving you to do 
the interesting stuff.

One advantage of Moose is the ability to mark class’ attributes 
as required, and apply typed constraints to those attributes. This 
means if you try to set an object’s attribute incorrectly, an 
exception is raised. Nice! However, a lot of developers new to Moose 
are soon daunted by the truck load of output that is thrown from 
Moose code. Here’s an example:

    Attribute (bar) is required at /home/pete/perl5/perlbrew/perls/perl-5.14.1/lib/site_perl/5.14.1/x86_64-linux/Moose/Meta/Attribute.pm line 510
        Moose::Meta::Attribute::initialize_instance_slot('Moose::Meta::Attribute=HASH(0x1e0e878)', 'Moose::Meta::Instance=HASH(0x1e14ec8)', 'Foo=HASH(0x1429b78)', 'HASH(0x1429170)') called at /home/pete/perl5/perlbrew/perls/perl-5.14.1/lib/site_perl/5.14.1/x86_64-linux/Class/MOP/Class.pm line 524
        Class::MOP::Class::_construct_instance('Moose::Meta::Class=HASH(0x1daf6e0)', 'HASH(0x1429170)') called at /home/pete/perl5/perlbrew/perls/perl-5.14.1/lib/site_perl/5.14.1/x86_64-linux/Class/MOP/Class.pm line 497
        Class::MOP::Class::new_object('Moose::Meta::Class=HASH(0x1daf6e0)', 'HASH(0x1429170)') called at /home/pete/perl5/perlbrew/perls/perl-5.14.1/lib/site_perl/5.14.1/x86_64-linux/Moose/Meta/Class.pm line 274
        Moose::Meta::Class::new_object('Moose::Meta::Class=HASH(0x1daf6e0)', 'HASH(0x1429170)') called at /home/pete/perl5/perlbrew/perls/perl-5.14.1/lib/site_perl/5.14.1/x86_64-linux/Moose/Object.pm line 28
        Moose::Object::new('Foo') called at /home/pete/perl_scratch/moose_error.pl line 13

Wow – what is all that? It’s actually a stack trace and is, in fact, 
nothing new to Perl. Usually exceptions thrown from Perl are 
reasonably terse but a stack trace can be provided using the confess 
function from the [Carp](https://metacpan.org/module/Carp) module. You can tell Moose to be a bit less 
verbose in its exceptions by telling it to use a different error handler:

    package Foo;

    use Moose;

    __PACKAGE__->meta->error_class('Moose::Error::Croak');

    ...

### Using Moose to validate data

Seeing as we can use typed constraints on our attributes in a Moose class, can we use Moose to validate our data? Why not?!

    package Address;

    use Moose;
    use Moose::Util::TypeConstraints;

    __PACKAGE__->meta->error_class('Moose::Error::Croak');

    subtype 'ISO3166_1'
        => as 'Str'
        => where { $_ =~ /^[a-z]{2}$/i }
        => message { 'Must be a 2 letter ISO code' };

    has line_1      => (is => 'rw', isa => 'Str', required => 1);
    has line_2      => (is => 'rw', isa => 'Str');
    has town        => (is => 'rw', isa => 'Str', required => 1);
    has county      => (is => 'rw', isa => 'Str');
    has postal_code => (is => 'rw', isa => 'Str', required => 1);
    has country_iso => (is => 'rw', isa => 'ISO3166_1', required => 1);

This is a reasonably simple class  – the only interesting bit is that 
we have set up a new subtype (see the [Moose manual on subtypes](https://metacpan.org/module/Moose::Manual::Types)) for 
a two letter country code to the ISO-3166-1 spec. It’s not as clever 
as it could be of course – it does not check the code is a valid one.

If we now try and instantiate an object using this class, with no 
parameters, we get an error:

    Attribute (line_1) is required at /home/pete/perl5/perlbrew/perls/perl-5.14.1/lib/site_perl/5.14.1/x86_64-linux/Moose/Meta/Attribute.pm line 51

Hmm. Ok, but what about all the other parameters that were required 
but we did not supply? Well, by default Moose just gives us the first 
error it finds in construction. Fortunately there are lots of helpful 
Moose extensions (in the MooseX namespace) on the CPAN and one of them, 
[MooseX::Constructor::AllErrors](https://metacpan.org/module/MooseX::Constructor::AllErrors), does what we need.  What it does is 
instead of returning a simple string as an exception, it throws an 
object containing all the constructor errors. You use it like so:

    package Address;

    use Moose;
    use Moose::Util::TypeConstraints;
    use MooseX::Constructor::AllErrors;

    subtype 'ISO3166_1'
    ...

Note **MooseX::Constructor::AllErrors** will set the meta object’s error 
class, so we don’t need to do it ourselves. Now we can catch the 
exception and deal with it, using the documented interface of 
[MooseX::Constructor::AllErrors::Error::Constructor](https://metacpan.org/module/MooseX::Constructor::AllErrors::Error::Constructor):

    use TryCatch;
    use 5.12.0;

    try {
        my $address = Address->new(country_iso => 'bleh');
    }
    catch (MooseX::Constructor::AllErrors::Error::Constructor $e) {
        for my $error ($e->errors) {
            say $error->attribute->name . ': ' . $error->message;
        }
    }

This prints:

    line_1: Attribute (line_1) is required
    town: Attribute (town) is required
    postal_code: Attribute (postal_code) is required
    country_iso: Attribute (country_iso) does not pass the type constraint because: Must be a 2 letter ISO code

So there you have it. Error messages from Moose don’t need to be a 
daunting ream of information  like a stack trace and can actually be 
turned into something less verbose or something more structured.

Have fun!

