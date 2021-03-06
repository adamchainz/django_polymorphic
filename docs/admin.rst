Django admin integration
========================

Off course, it's possible to register individual polymorphic models in the Django admin interface.
However, to use these models in a single cohesive interface, some extra base classes are available.

The polymorphic admin interface works in a simple way:

* The add screen gains an additional step where the desired child model is selected.
* The edit screen displays the admin interface of the child model.
* The list screen still displays all objects of the base class.

The polymorphic admin is implemented via a parent admin that forwards the *edit* and *delete* views
to the ``ModelAdmin`` of the derived child model. The *list* page is still implemented by the parent model admin.

Both the parent model and child model need to have a ``ModelAdmin`` class.

The parent model
----------------

The parent model needs to inherit ``PolymorphicParentModelAdmin``, and implement the following:

* ``base_model`` should be set
* ``child_models`` or ``get_child_models()`` should return an iterable of Model classes.

The exact implementation can depend on the way your module is structured.
For simple inheritance situations, ``child_models`` is the best solution.
For large applications, ``get_child_models()`` can be used to query a plugin registration system.

By default, the non_polymorphic() method will be called on the queryset, so
only the Parent model will be provided to the list template.  This is to avoid
the performance hit of retrieving child models.

This can be controlled by setting the ``polymorphic_list`` property on the
parent admin.  Setting it to True will provide child models to the list template.

If you use other applications such as django-reversion_ or django-mptt_, please check +:ref:`third-party`.

Note: If you are using non-integer primary keys in your model, you have to edit ``pk_regex``, 
for example ``pk_regex = '([\w-]+)'`` if you use UUIDs. Otherwise you cannot change model entries.

The child models
----------------

The admin interface of the derived models should inherit from ``PolymorphicChildModelAdmin``.
Again, ``base_model`` should be set in this class as well.
This class implements the following features:

* It corrects the breadcrumbs in the admin pages.
* It extends the template lookup paths, to look for both the parent model and child model in the ``admin/app/model/change_form.html`` path.
* It allows to set ``base_form`` so the derived class will automatically include other fields in the form.
* It allows to set ``base_fieldsets`` so the derived class will automatically display any extra fields.
* Although it must be registered with admin site, by default it's hidden from admin site index page.
  This can be overriden by adding ``show_in_index = True`` in admin class.

The standard ``ModelAdmin`` attributes ``form`` and ``fieldsets`` should rather be avoided at the base class,
because it will hide any additional fields which are defined in the derived model. Instead,
use the ``base_form`` and ``base_fieldsets`` instead. The ``PolymorphicChildModelAdmin`` will
automatically detect the additional fields that the child model has, display those in a separate fieldset.


Polymorphic Inlines
-------------------

To add a polymorphic child model as an Inline for another model, add a field to the inline's readonly_fields list formed by the lowercased name of the polymorphic parent model with the string "_ptr" appended to it. Otherwise, trying to save that model in the admin will raise an AttributeError with the message "can't set attribute".


.. _admin-example:

Example
-------

The models are taken from :ref:`advanced-features`.

.. code-block:: python

    from django.contrib import admin
    from polymorphic.admin import PolymorphicParentModelAdmin, PolymorphicChildModelAdmin, PolymorphicChildModelFilter
    from .models import ModelA, ModelB, ModelC, StandardModel


    class ModelAChildAdmin(PolymorphicChildModelAdmin):
        """ Base admin class for all child models """
        base_model = ModelA

        # By using these `base_...` attributes instead of the regular ModelAdmin `form` and `fieldsets`,
        # the additional fields of the child models are automatically added to the admin form.
        base_form = ...
        base_fieldsets = (
            ...
        )


    class ModelBAdmin(ModelAChildAdmin):
        base_model = ModelB
        # define custom features here


    class ModelCAdmin(ModelBAdmin):
        base_model = ModelC
        show_in_index = True  # makes child model admin visible in main admin site
        # define custom features here


    class ModelAParentAdmin(PolymorphicParentModelAdmin):
        """ The parent model admin """
        base_model = ModelA
        child_models = (ModelB, ModelC)
        list_filter = (PolymorphicChildModelFilter,)  # This is optional.


    class ModelBInline(admin.StackedInline):
        model = ModelB
        fk_name = 'modelb'
        readonly_fields = ['modela_ptr']
    
        
    class StandardModelAdmin(admin.ModelAdmin):
        inlines = [ModelBInline]
        

    # Only the parent needs to be registered:
    admin.site.register(ModelA, ModelAParentAdmin)
    admin.site.register(ModelB, ModelBAdmin)
    admin.site.register(ModelC, ModelCAdmin)
    admin.site.register(StandardModel, StandardModelAdmin)


Filtering child types
---------------------

Child model types can be filtered by adding a :class:`~polymorphic.admin.PolymorphicChildModelFilter`
to the ``list_filter`` attribute. See the example above.

.. _django-reversion: https://github.com/etianen/django-reversion
.. _django-mptt: https://github.com/django-mptt/django-mptt
