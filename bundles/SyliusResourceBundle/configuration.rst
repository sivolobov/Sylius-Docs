Configuring your bundle
=======================

Creating a XmlMappingDriver
---------------------------

.. code-block:: php

    class MyBundle extends AbstractResourceBundle
    {
        // You need to specify a prefix for your bundle
        protected function getBundlePrefix()
        {
            return 'app_bundle_name';
        }

        // You need specify the namespace where are stored your models
        protected function getModelNamespace()
        {
            return 'MyApp\MyBundle\Model';
        }

        // You can specify the path where are stored the doctrine mapping, by default this method will returns
        // model. This path is relative to the Resources/config/doctrine/.
        protected function getDoctrineMappingDirectory()
        {
            return 'model';
        }
    }


Using the ResolveDoctrineTargetEntitiesPass
-------------------------------------------

.. code-block:: php

    class MyBundle extends AbstractResourceBundle
    {
        // You need to specify a prefix for your bundle
        protected function getBundlePrefix()
        {
            return 'app_bundle_name';
        }

        // You need to specify the mapping between your intefaces and your models. Like the following example you can
        // get the classname of your model in the container (See the following chapater for more informations).
        protected function getModelInterfaces()
        {
            return array(
                'MyApp\MyBundle\ModelInterface' => 'sylius.model.resource.class',
            );
        }
    }

Configuring your resources
==========================

There are two ways to configure the resources used by this bundle. You can manage your configuration for all yours bundles (explained in Basic Configuration) or into yours bundles (explained in Advanced configuration).

Basic configuration
-------------------

In your `app/config.yml` (or in an imported configuration file), you need to define what resources you want to use :

.. code-block:: yaml

    sylius_resource:
        resources:
            my_app.entity_key:
                driver: doctrine/orm
                templates: App:User
                classes:
                    model: MyApp\Entity\EntityName
                    interface: MyApp\Entity\EntityKeyInterface
                    controller: Sylius\Bundle\ResourceBundle\Controller\ResourceController
                    repository: Sylius\Bundle\ResourceBundle\Doctrine\ORM\EntityRepository
            my_app.other_entity_key:
                driver: doctrine/odm
                templates: App:User
                classes:
                    model: MyApp\Entity\OtherEntityKey
                    interface: MyApp\Document\OtherEntityKeyInterface
                    controller: Sylius\Bundle\ResourceBundle\Controller\ResourceController
                    repository: Sylius\Bundle\ResourceBundle\Doctrine\ODM\DocumentRepository

At this step:

.. code-block:: bash

    $ php app/console container:debug | grep entity_key
    my_app.repository.entity_key   container MyApp\Entity\EntityName
    my_app.controller.entity_key   container Sylius\Bundle\ResourceBundle\Controller\ResourceController
    my_app.repository.entity_key   container Sylius\Bundle\ResourceBundle\Doctrine\ORM\EntityRepository
    //...

    $ php app/console container:debug --parameters | grep entity_key
    sylius.config.classes        {"my_app.entity_key": {"driver":"...", "classes":{"model":"...", "controller":"...", "repository":"...", "interface":"..."}}}
    //...

Advanced configuration
----------------------

.. note::

    Since the 0.11 your bundle class must implement `ResourceBundleInterface`, you must list the supported doctrine driver.
    The available drivers are SyliusResourceBundle::DRIVER_DOCTRINE_ORM, SyliusResourceBundle::DRIVER_DOCTRINE_MONGODB_ODM or SyliusResourceBundle::DRIVER_DOCTRINE_PHPCR_ODM

.. code-block:: php

    class MyBundle extends AbstractResourceBundle
    {
        public static function getSupportedDrivers()
        {
            return array(
                SyliusResourceBundle::DRIVER_DOCTRINE_ORM
            );
        }
    }

You need to expose a semantic configuration for your bundle. The following example show you a basic `Configuration` that the resource bundle needs to work.

.. code-block:: php

    class Configuration implements ConfigurationInterface
    {
        public function getConfigTreeBuilder()
        {
            $treeBuilder = new TreeBuilder();
            $rootNode = $treeBuilder->root('bundle_name');

            $rootNode
                // Driver used by the resource bundle
                ->children()
                    ->scalarNode('driver')->isRequired()->cannotBeEmpty()->end()
                ->end()

                // Validation groups used by the form component
                ->children()
                    ->arrayNode('validation_groups')
                        ->addDefaultsIfNotSet()
                        ->children()
                            ->arrayNode('MyEntity')
                                ->prototype('scalar')->end()
                                ->defaultValue(array('your_group'))
                            ->end()
                        ->end()
                    ->end()
                ->end()

                // The resources
                ->children()
                    ->arrayNode('classes')
                        ->addDefaultsIfNotSet()
                        ->children()
                            ->arrayNode('my_entity')
                                ->addDefaultsIfNotSet()
                                ->children()
                                    ->scalarNode('model')->defaultValue('MyApp\MyCustomBundle\Model\MyEntity')->end()
                                    ->scalarNode('controller')->defaultValue('Sylius\Bundle\ResourceBundle\Controller\ResourceController')->end()
                                    ->scalarNode('repository')->end()
                                    ->scalarNode('form')->defaultValue('MyApp\MyCustomBundle\Form\Type\MyformType')->end()
                                ->end()
                            ->end()
                            ->arrayNode('my_other_entity')
                                ->addDefaultsIfNotSet()
                                ->children()
                                    ->scalarNode('model')->defaultValue('MyApp\MyCustomBundle\Model\MyOtherEntity')->end()
                                    ->scalarNode('controller')->defaultValue('Sylius\Bundle\ResourceBundle\Controller\ResourceController')->end()
                                    ->scalarNode('form')->defaultValue('MyApp\MyCustomBundle\Form\Type\MyformType')->end()
                                ->end()
                            ->end()
                        ->end()
                    ->end()
                ->end()
            ;

            return $treeBuilder;
        }
    }

The resource bundle provide you `AbstractResourceExtension`, your bundle extension have to extends it.

.. code-block:: php

    use Sylius\Bundle\ResourceBundle\DependencyInjection\AbstractResourceExtension;

    class MyBundleExtension extends AbstractResourceExtension
    {
        // You can choose your application name, it will use to prefix the configuration keys in the container (the default value is sylius).
        protected $applicationName = 'my_app';

        // You can define where yours service definitions are
        protected $configDirectory = '/../Resources/config';

        // You can define what service definitions you want to load
        protected $configFiles = array(
            'services',
            'forms',
        );

        public function load(array $config, ContainerBuilder $container)
        {
            $this->configure(
                $config,
                new Configuration(),
                $container,
                self::CONFIGURE_LOADER | self::CONFIGURE_DATABASE | self::CONFIGURE_PARAMETERS | self::CONFIGURE_VALIDATORS
            );
        }
    }

The last parameter of the `AbstractResourceExtension::configure()` allows you to define what functionalities you want to use :

 * CONFIGURE_LOADER : load yours service definitions located in `$applicationName`
 * CONFIGURE_PARAMETERS : set to the container the configured resource classes using the pattern `my_app.serviceType.resourceName.class`
   For example : `sylius.controller.product.class`. For a form, it is a bit different : 'sylius.form.type.product.class'
 * CONFIGURE_VALIDATORS : set to the container the configured validation groups using the pattern `my_app.validation_group.modelName`
   For example `sylius.validation_group.product`
 * CONFIGURE_DATABASE : Load the database driver, available drivers are `doctrine/orm`, `doctrine/mongodb-odm` and `doctrine/phpcr-odm`

At this step:

.. code-block:: bash

    $ php app/console container:debug | grep my_entity
    my_app.controller.my_entity              container Sylius\Bundle\ResourceBundle\Controller\ResourceController
    my_app.form.type.my_entity               container MyApp\MyCustomBundle\Form\Type\TaxonomyType
    my_app.manager.my_entity                 n/a       alias for doctrine.orm.default_entity_manager
    my_app.repository.my_entity              container MyApp\MyCustomer\ModelRepository
    //...

    $ php app/console container:debug --parameters | grep my_entity
    my_app.config.classes                   {...}
    my_app.controller.my_entity.class       MyApp\MyCustomBundle\ModelController
    my_app.form.type.my_entity.class        MyApp\MyCustomBundle\FormType
    my_app.model.my_entity.class            MyApp\MyCustomBundle\Model
    my_app.repository.my_entity.class       MyApp\MyCustomBundle\ModelRepository
    my_app.validation_group.my_entity       ["my_app"]
    my_app_my_entity.driver                 doctrine/orm
    my_app_my_entity.driver.doctrine/orm    true
    //...

You can overwrite the configuration of your bundle like that :

.. code-block:: php

    bundle_name:
        driver: doctrine/orm
        validation_groups:
            product: [myCustomGroup]
        classes:
            my_entity:
                model: MyApp\MyOtherCustomBundle\Model
                controller: MyApp\MyOtherCustomBundle\Entity\ModelController
                repository: MyApp\MyOtherCustomBundle\Repository\ModelRepository
                form: MyApp\MyOtherCustomBundle\Form\Type\FormType


Combining the both configurations
---------------------------------

For now, with the advanced configuration you can not use serveral drivers but they can be overwritten. Example, you want to use
`doctrine/odm` for `my_other_entity` (see previous chapter), you just need to add this extra configuration to the `app/config.yml`.

.. code-block:: yaml

    sylius_resource:
        resources:
            my_app.other_entity_key:
                driver: doctrine/odm
                classes:
                    model: %my_app.model.my_entity.class%

And your manager will be overwrite:

.. code-block:: bash

    $ php app/console container:debug | grep my_app.manager.other_entity_key
    my_app.manager.other_entity_key       n/a       alias for doctrine.odm.default_entity_manager

And... we're done!