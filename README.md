# [![](https://bitbag.io/wp-content/uploads/2021/04/ShippingExportPlugin.png)](https://bitbag.io/contact-us/?utm_source=github&utm_medium=referral&utm_campaign=plugins_shipping_export)

# BitBag SyliusShippingExportPlugin

----

[![](https://img.shields.io/packagist/l/bitbag/shipping-export-plugin.svg)](https://packagist.org/packages/bitbag/shipping-export-plugin "License") 
[![](https://img.shields.io/packagist/v/bitbag/shipping-export-plugin.svg)](https://packagist.org/packages/bitbag/shipping-export-plugin "Version")
[![](https://img.shields.io/github/workflow/status/BitBagCommerce/SyliusShippingExportPlugin/Build)](https://github.com/BitBagCommerce/SyliusShippingExportPlugin/actions "Build status")
[![](https://img.shields.io/scrutinizer/quality/g/BitBagCommerce/SyliusShippingExportPlugin.svg)](https://scrutinizer-ci.com/g/BitBagCommerce/SyliusShippingExportPlugin/ "Scrutinizer") 
[![](https://poser.pugx.org/bitbag/shipping-export-plugin/downloads)](https://packagist.org/packages/bitbag/shipping-export-plugin "Total Downloads")
[![Slack](https://img.shields.io/badge/community%20chat-slack-FF1493.svg)](http://sylius-devs.slack.com)
[![Support](https://img.shields.io/badge/support-contact%20author-blue])](https://bitbag.io/contact-us/?utm_source=github&utm_medium=referral&utm_campaign=plugins_shipping_export)

<p>
 <img src="https://sylius.com/assets/badge-approved-by-sylius.png" width="85">
</p> 

At BitBag we do believe in open source. However, we are able to do it just because of our awesome clients, who are kind enough to share some parts of our work with the community. Therefore, if you feel like there is a possibility for us working together, feel free to reach us out. You will find out more about our professional services, technologies and contact details at [https://bitbag.io/](https://bitbag.io/contact-us/?utm_source=github&utm_medium=referral&utm_campaign=plugins_shipping_export).

## Table of Content

***

* [Overview](#overview)
* [Installation](#installation)
* [Usage](#usage)
* [About us](#about-us)
  * [Community](#community)
* [Demo](#demo-sylius-shop)
* [License](#license)
* [Contact](#contact)

# Overview

***

Managing shipments in any eCommerce app is something that may be tricky. There are many shipping providers and
each has its own API format you might want to use to export shipping data and request the pickup. To make this process more simple and generic, we decided to create an abstract layer for Sylius platform based applications for this purpose. This plugin allows you to write simple API calls and configuration form for
specific shipping provider. The workflow is quite simple - configure a proper data that's needed to export a shipment, like access key or pickup hour,
book a courier for an order with one click and get shipping label file if any was received from the API.
The implementation limits to writing a shipping provider gateway configuration form, one event listener and webservice access layer.

If you are curious about the details of this plugin, read [this blog post](https://bitbag.shop/blog/bitbag-shipping-export-plugin-simple-way-to-control-shipments-in-your-online-store).

## Installation

### Requirements

We work on stable, supported and up-to-date versions of packages. We recommend you to do the same.

| Package    | Version      |
|:----------:|:-------------|
|PHP         |  ^7.4        |
|Sylius      |  ^1.8        |
---

```bash
$ composer require bitbag/shipping-export-plugin
```

Add plugin dependencies to your `config/bundles.php` file:

```php
return [
    ...

    BitBag\SyliusShippingExportPlugin\BitBagSyliusShippingExportPlugin::class => ['all' => true],
];
```

Import required config in your `config/packages/_sylius.yaml` file:
```yaml
# config/packages/_sylius.yaml

imports:
    ...

    - { resource: "@BitBagSyliusShippingExportPlugin/Resources/config/config.yml" }
```

Import routing in your config/routes.yaml file:

```yaml
# config/routes.yaml

bitbag_shipping_export_plugin:
    resource: "@BitBagSyliusShippingExportPlugin/Resources/config/routing.yml"
    prefix: /admin
```

## Usage

### Adding shipping export configuration form

```php
namespace App\Form\Type;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Validator\Constraints\NotBlank;

final class FrankMartinShippingGatewayType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('iban', TextType::class, [
                'label' => 'IBAN',
                'constraints' => [
                    new NotBlank([
                        'message' => 'IBAN number cannot be blank.',
                        'groups' => 'bitbag',
                    ]),
                ],
            ])
            ->add('address', TextType::class, [
                'label' => 'Address',
                'constraints' => [
                    new NotBlank([
                        'message' => 'Address cannot be blank.',
                        'groups' => 'bitbag',
                    ]),
                ],
            ])
        ;
    }
}
```

#### Service definition
```yaml
services:
    app.form.type.frank_martin_shipping_gateway:
        class: App\Form\Type\FrankMartinShippingGatewayType
        tags:
            - { name: bitbag.shipping_gateway_configuration_type, type: "frank_martin_shipping_gateway", label: "Transporter Gateway" }
```

### Adding shipping export event listener
```php
namespace App\EventListener;

use BitBag\SyliusShippingExportPlugin\Entity\ShippingExportInterface;
use Doctrine\Persistence\ObjectManager;
use Sylius\Bundle\ResourceBundle\Event\ResourceControllerEvent;
use Symfony\Component\Filesystem\Filesystem;
use Symfony\Component\HttpFoundation\Session\Flash\FlashBagInterface;
use Webmozart\Assert\Assert;

final class FrankMartinShippingExportEventListener
{
    /** @var FlashBagInterface */
    private $flashBag;

    /** @var Filesystem */
    private $filesystem;

    /** @var ObjectManager */
    private $shippingExportManager;

    /** @var string */
    private $shippingLabelsPath;

    public function __construct(
        FlashBagInterface $flashBag,
        Filesystem $filesystem,
        ObjectManager $shippingExportManager,
        string $shippingLabelsPath
    ) {
        $this->flashBag = $flashBag;
        $this->filesystem = $filesystem;
        $this->shippingExportManager = $shippingExportManager;
        $this->shippingLabelsPath = $shippingLabelsPath;
    }

    public function exportShipment(ResourceControllerEvent $event): void
    {
        /** @var ShippingExportInterface $shippingExport */
        $shippingExport = $event->getSubject();
        Assert::isInstanceOf($shippingExport, ShippingExportInterface::class);

        $shippingGateway = $shippingExport->getShippingGateway();
        Assert::notNull($shippingGateway);

        if ('frank_martin_shipping_gateway' !== $shippingGateway->getCode()) {
            return;
        }

        if (false) {
            $this->flashBag->add('error', 'bitbag.ui.shipping_export_error'); // Add an error notification

            return;
        }

        $this->flashBag->add('success', 'bitbag.ui.shipment_data_has_been_exported'); // Add success notification
        $this->saveShippingLabel($shippingExport, 'Some label content received from external API', 'pdf'); // Save label
        $this->markShipmentAsExported($shippingExport); // Mark shipment as "Exported"
    }

    public function saveShippingLabel(
        ShippingExportInterface $shippingExport,
        string $labelContent,
        string $labelExtension
    ): void {
        $labelPath = $this->shippingLabelsPath
            . '/' . $this->getFilename($shippingExport)
            . '.' . $labelExtension;

        $this->filesystem->dumpFile($labelPath, $labelContent);
        $shippingExport->setLabelPath($labelPath);

        $this->shippingExportManager->persist($shippingExport);
        $this->shippingExportManager->flush();
    }

    private function getFilename(ShippingExportInterface $shippingExport): string
    {
        $shipment = $shippingExport->getShipment();
        Assert::notNull($shipment);

        $order = $shipment->getOrder();
        Assert::notNull($order);

        $orderNumber = $order->getNumber();

        $shipmentId = $shipment->getId();

        return implode(
            '_',
            [
                $shipmentId,
                preg_replace('~[^A-Za-z0-9]~', '', $orderNumber),
            ]
        );
    }

    private function markShipmentAsExported(ShippingExportInterface $shippingExport): void
    {
        $shippingExport->setState(ShippingExportInterface::STATE_EXPORTED);
        $shippingExport->setExportedAt(new \DateTime());

        $this->shippingExportManager->persist($shippingExport);
        $this->shippingExportManager->flush();
    }
}
```

#### Service definition
```yaml
services:
    app.event_listener.frank_martin_shipping_export:
        class: App\EventListener\FrankMartinShippingExportEventListener
        arguments:
            - '@session.flash_bag'
            - '@filesystem'
            - '@bitbag.manager.shipping_export'
            - '%bitbag.shipping_labels_path%'
        tags:
            - { name: kernel.event_listener, event: 'bitbag.shipping_export.export_shipment', method: exportShipment }
```

#### Plugin parameters
```yaml
parameters:
    bitbag.shipping_gateway.validation_groups: ['bitbag']
    bitbag.shipping_labels_path: '%kernel.project_dir%/shipping_labels'
```

### Available services you can [decorate](https://symfony.com/doc/current/service_container/service_decoration.html) and forms you can [extend](http://symfony.com/doc/current/form/create_form_type_extension.html)
```bash
$ bin/console debug:container | grep bitbag
```

### Parameters you can override in your parameters.yml(.dist) file
```bash
$ bin/console debug:container --parameters | grep bitbag
```

## Testing
```bash
$ composer install
$ APP_ENV=test symfony server:start --port=8080 --dir=tests/Application/public --daemon
$ cd tests/Application
$ yarn install
$ yarn run gulp
$ bin/console assets:install public -e test
$ bin/console doctrine:schema:create -e test
$ shipping-export
$ open http://localhost:8080
$ vendor/bin/behat
$ vendor/bin/phpspec run
$ vendor/bin/phpstan analyse -c phpstan.neon -l max src/
$ vendor/bin/ecs check src
```

## We are here to help
This **open-source plugin was developed to help the Sylius community**. If you have any additional questions, would like help with installing or configuring the plugin or need any assistance with your Sylius project - let us know!

[![](https://bitbag.io/wp-content/uploads/2020/10/button-contact.png)](https://bitbag.io/contact-us/?utm_source=github&utm_medium=referral&utm_campaign=plugins_shipping_export)

Read more about  [BitBag Sylius Shipping Export Plugin](https://bitbag.io/bitbag-shipping-export-plugin-simple-way-to-control-shipments-in-your-online-store/?utm_source=github&utm_medium=referral&utm_campaign=plugins_shipping_export)
# About us

---

BitBag is an agency that provides high-quality **eCommerce and Digital Experience software**. Our main area of expertise includes eCommerce consulting and development for B2C, B2B, and Multi-vendor Marketplaces.
The scope of our services related to Sylius includes:
- **Consulting** in the field of strategy development
- Personalized **headless software development**
- **System maintenance and long-term support**
- **Outsourcing**
- **Plugin development**
- **Data migration**

Some numbers regarding Sylius:
* **20+ experts** including consultants, UI/UX designers, Sylius trained front-end and back-end developers,
* **100+ projects** delivered on top of Sylius,
* Clients from  **20+ countries**
* **3+ years** in the Sylius ecosystem.

---

If you need some help with Sylius development, don't be hesitate to contact us directly. You can fill the form on [this site](https://bitbag.io/contact-us/?utm_source=github&utm_medium=referral&utm_campaign=plugins_shipping_export) or send us an e-mail to hello@bitbag.io!

---

[![](https://bitbag.io/wp-content/uploads/2020/10/badges-sylius.png)](https://bitbag.io/contact-us/?utm_source=github&utm_medium=referral&utm_campaign=plugins_shipping_export)

## Community

---- 

For online communication, we invite you to chat with us & other users on [Sylius Slack](https://sylius-devs.slack.com/).

# Demo Sylius shop

---

We created a demo app with some useful use-cases of plugins!
Visit [sylius-demo.bitbag.io](https://sylius-demo.bitbag.io/) to take a look at it. The admin can be accessed under
[sylius-demo.bitbag.io/admin/login](https://sylius-demo.bitbag.io/admin/login) link and `sylius: sylius` credentials.
Plugins that we have used in the demo:

| BitBag's Plugin | GitHub | Sylius' Store|
| ------ | ------ | ------|
| ACL Plugin | *Private. Available after the purchasing.*| https://plugins.sylius.com/plugin/access-control-layer-plugin/|
| Braintree Plugin | https://github.com/BitBagCommerce/SyliusBraintreePlugin |https://plugins.sylius.com/plugin/braintree-plugin/|
| CMS Plugin | https://github.com/BitBagCommerce/SyliusCmsPlugin | https://plugins.sylius.com/plugin/cmsplugin/|
| Elasticsearch Plugin | https://github.com/BitBagCommerce/SyliusElasticsearchPlugin | https://plugins.sylius.com/plugin/2004/|
| Mailchimp Plugin | https://github.com/BitBagCommerce/SyliusMailChimpPlugin | https://plugins.sylius.com/plugin/mailchimp/ |
| Multisafepay Plugin | https://github.com/BitBagCommerce/SyliusMultiSafepayPlugin |
| Wishlist Plugin | https://github.com/BitBagCommerce/SyliusWishlistPlugin | https://plugins.sylius.com/plugin/wishlist-plugin/|
| **Sylius' Plugin** | **GitHub** | **Sylius' Store** |
| Admin Order Creation Plugin | https://github.com/Sylius/AdminOrderCreationPlugin | https://plugins.sylius.com/plugin/admin-order-creation-plugin/ |
| Invoicing Plugin | https://github.com/Sylius/InvoicingPlugin | https://plugins.sylius.com/plugin/invoicing-plugin/ |
| Refund Plugin | https://github.com/Sylius/RefundPlugin | https://plugins.sylius.com/plugin/refund-plugin/ |

**If you need an overview of Sylius' capabilities, schedule a consultation with our expert.**
**If you need an overview of this plugin, schedule a consultation with our expert.**

[![](https://bitbag.io/wp-content/uploads/2020/10/button_free_consulatation-1.png)](https://bitbag.io/contact-us/?utm_source=github&utm_medium=referral&utm_campaign=plugins_shipping_export)

## Additional resources for developers

---
To learn more about our contribution workflow and more, we encourage ypu to use the following resources:
* [Sylius Documentation](https://docs.sylius.com/en/latest/)
* [Sylius Contribution Guide](https://docs.sylius.com/en/latest/contributing/)
* [Sylius Online Course](https://sylius.com/online-course/)

## License

---

This plugin's source code is completely free and released under the terms of the MIT license.

[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen.)

## Contact

---
If you want to contact us, the best way is to fill the form on [our website](https://bitbag.io/contact-us/?utm_source=github&utm_medium=referral&utm_campaign=plugins_shipping_export) or send us an e-mail to hello@bitbag.io with your question(s). We guarantee that we answer as soon as we can!

[![](https://bitbag.io/wp-content/uploads/2020/10/footer.png)](https://bitbag.io/contact-us/?utm_source=github&utm_medium=referral&utm_campaign=plugins_shipping_export)
