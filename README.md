#dry-rest-permissions


[<img src="https://api.travis-ci.org/Helioscene/dry-rest-permissions.svg?branch=master">](http://travis-ci.org/Helioscene/dry-rest-permissions?branch=master) [<img src="https://img.shields.io/pypi/v/dry-rest-permissions.svg">](https://pypi.python.org/pypi/dry-rest-permissions)

##Overview

Rules based permissions for the Django Rest Framework.

This framework is a perfect fit for apps that have many tables and relationships between them. It provides a framework that allows you to define, for each action or groups of actions, what users have permission based on existing data in your database.

##What does DRY Rest Permissions provide?

1. A framework for defining for defining global and object level permissions per action.
  a. Support for broadly defining permissions by grouping actions into safe and unsafe types.
  b. Support for defining only global (table level) permissions or only object (row level) permissions.
  c. Support for custom list and detail actions.
2. A serializer field that will return permissions for an object to your client app. This is DRY and works with your existing permission definitions.
3. A framework for limiting list requests based on permissions
  a. Support for custom list actions
  
##Why is DRY Rest Permissions different than other DRF permission packages?

Most other DRF permissions are based on django-guardian. Django-gaurdian is an explicit approach to permissions that requires data to be saved in tables that explicitly grants permissions for certain actions. For apps that have many ways for a user to be given permission to certain actions, this approach can be very hard to maintain.

For example: you may have an app which lets you create and modify projects if you are an admin of an association that owns the project. This means that a user's permission will be granted or revoked based on many possabilities including ownership of the project transering to a different association, the user's admin status in the association changing and the user entering or leaving the association. This would need a lot of triggers that would key off of these actions and explicitly change permissions.

DRY Rest Permissions allows developers to easily describe what gives someone permission using the current data in an implicit way.

##Requirements

-  Python (2.7)
-  Django (1.7, 1.8)
-  Django REST Framework (3.0, 3.1)

##Installation

Install using ``pip``…

    $ pip install dry-rest-permissions

##Setup

Add to INSTALLED_APPS

    INSTALLED_APPS = (
      ...
      'dry_rest_permissions',)

##Global vs. Object Permissions
DRY Rest Permissions allows you to define both global and object level permissions.

Global permissions are always checked first and define the ability of a user to take an action on an entire model. For example you can define whether a user has the ability to update any projects from the database.

Object permissions are checked if global permissions pass and define whether a user has the ability to perform a specific action on a single object. These are also known as row level permissions.

##Read/Write permissions vs. Specific Actions
DRY Rest Permissions allows you to define permissions for both the standard actions (``list``, ``retrieve``, ``update``, ``destroy`` and ``create``) and custom actions defined using ``@detail_route`` and ``@list_route``.

If you don't need to define permissions on a granular action level you can generally define read or write permissions for a model. Read groups together list and retrieve, while write groups together destroy, update and create. All custom actions that use ``GET`` methods are considered read actions and all other methods are considered write.

Specific action permissions take precedence over general read or write permissions. For example you can lock down write permissions by always returning ``False`` and open up just update permissions for certain users.

The ``partial_update`` action is also supported, but by default is grouped with the update permission.

##Add permissions to an API

Permissions can be added to ModelViewSet based APIs.

###Add permission class to a ModelViewSet

First you must add ``DRYPermissions`` to the viewset's ``permission_classes``

    from rest_framework import viewsets
    from dry_rest_permissions.generics import DRYPermissions
    
    class ProjectViewSet(viewsets.ModelViewSet):
      permission_classes = (DRYPermissions,)
      queryset = Project.objects.all()
      serializer_class = ProjectSerializer

You may also use ``DRYGlobalPermissions`` and ``DRYObjectPermissions``, which will only check global or object permissions.

###Define permission logic on the model
Permissions for DRY Rest permissions are defined on the model so that they can be accessed both from the view for checking and from the serializer for display purposes with the ``DRYPermissionsField``.

**Gloabl permissions** are defined as either ``@staticmethod`` or ``@classmethod`` methods with the format ``has_<action/read/write>_permission``.

**Object permissions** are defined as methods with the format ``has_object_<action/read.write>_permission``.

The following example shows how you would allow all users to read and create projects, while locking down the ability for any user to perform any other write action. In the example, read global and object permissions return True, which grants permission to those actions. Write, globally returns False, which locks down write actions. However, create is a specific action and therefore takes precedence over write and gives all users the ability to create projects.

    from django.db import models
    from django.contrib.auth.models import User
    
    class Project(models.Model):
      owner = models.ForeignKey('User')
      
      @staticmethod
      def has_read_permission(request):
        return True
        
      def has_object_read_permission(self, request):
        return True
        
      @staticmethod
      def has_write_permission(request):
        return False
        
      @staticmethod
      def has_create_permission(request):
        return True
  
  Now we will add to this example and allow project owners to update or destroy a project.
  
    class Project(models.Model):
      owner = models.ForeignKey('User')
      ...
        
      @staticmethod
      def has_write_permission(request):
        """
        We can remove the has_create_permission because this implicitly grants that permission.
        """
        return True
        
      def has_object_write_permission(self, request):
        if request.user == self.owner:
          return True
        return False
  
  If we just wanted to grant update permission, but not destroy we could do this:
  
      class Project(models.Model):
      owner = models.ForeignKey('User')
      ...
        
      @staticmethod
      def has_write_permission(request):
        """
        We can remove the has_create_permission because this implicitly grants that permission.
        """
        return True
        
      def has_object_write_permission(self, request):
        return False
        
      def has_object_update_permission(self, request):
        if request.user == self.owner:
          return True
        return False

###Custom action permissions
If a custom action, ``publish``, were created using ``@detail_route`` then permissions could be defined like so:

    class Project(models.Model):
      owner = models.ForeignKey('User')
      ...
        
      @staticmethod
      def has_publish_permission(request):
        return True
        
      def has_object_publish_permission(self, request):
        if request.user == self.owner:
          return True
        return False
  
###Helpful decorators
Three decorators were defined for common permission checks
``@allow_staff_or_superuser`` - Allows any user that has staff or superuser status to have the permission.
``@authenticated_users`` - This permission will only be checked for authenticated users. Unauthenticated users will automatically be denied permission.
``@unauthenticated_users`` - This permission will only be checked for unauthenticated users. Authenticated users will automatically be denied permission.

Example:

    from dry_rest_permissions.generics import allow_staff_or_superuser, authenticated_users
    class Project(models.Model):
      owner = models.ForeignKey('User')
      ...
        
      @staticmethod
      @authenticated_users
      def has_publish_permission(request):
        return True
        
      @allow_staff_or_sueruser
      def has_object_publish_permission(self, request):
        if request.user == self.owner:
          return True
        return False

##Returning Permissions to the Client App
You often need to know all of the possible permissions that are available to the current user from within your client app so that you can show certain create, edit and destroy options. Sometimes you need to know the permssions on the client app so that you can display messages to them. ``DRYPermissionsField`` allows you to return these permissions in a serializer without having to redefine your permission logic. DRY!

    from dry_rest_permissions.generics import DRYPermissionsField
    
    class ProjectSerializer(serializers.ModelSerializer):
      permissions = DRYPermissionsField()
      
      class Meta:
          model = Project
          fields = ('owner', 'permissions')
  