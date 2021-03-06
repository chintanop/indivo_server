Indivo Transforms
=================

Introduction
------------

For the pipeline to be functional, data must be transformed from its original format into processed medical facts ready
to be stored in the database. Each schema in Indivo therefore defines a :term:`transform` that can be applied to any 
document that validates against the schema.

.. _transform-output-types:

Transform Outputs
-----------------

The ultimate output of the transformation step in the data pipeline is a set of :term:`Fact objects <Fact>` ready for 
storage in the database. However, technologies like XSLT are incapable of producing python objects as output. We looked
around for a simple, standard way of modeling data that would meet our needs, and came up empty (though we're open to
suggestions if you think you have the silver bullet). As a result, we've created our own language, 
:doc:`sdm`, to both define our data models and represent documents (in XML or JSON) that match them.

Thus, transforms may output data in any of the following formats:

* :ref:`Simple Data Model JSON (SDMJ) <sdmj>`
* :ref:`Simple Data Model XML (SDMX) <sdmx>`
* Python Fact objects

Outputs are validated on a per-datamodel-basis. For data model definitions and example outputs, 
see :doc:`data-models/index`.


Types of Transforms
-------------------

Indivo currently accepts Transforms in two formats:

* XSLT documents
* Python classes

This may change as Indivo begins to accept data in more, varied formats.

XSLTs
^^^^^

We won't cover XSLTs in any detail here, as their format and use is clearly outlined 
`in the specification <http://www.w3.org/TR/xslt>`_. Since XSLT is traditionally used to transform XML to XML, the
most natural output format for XSLTs is :ref:`SDMX <sdmx>`.

.. _python-transforms:

Python
^^^^^^

For those unskilled in the arts of XSLT, we also allow transforms to be defined using python. To define a
transform, simply subclass :py:class:`indivo.document_processing.BaseTransform` and define a valid transformation
method. Valid methods are:

.. automethod:: indivo.document_processing.BaseTransform.to_facts

   As an example, here's how you might implement this for a transform from our 
   :doc:`Simple Clinical Note Schema <schemas/scn-schema>` to our :doc:`Simple Clinical Note Data Model <data-models/scn>`::

     from indivo.document_processing import BaseTransform
     from indivo.models import SimpleClinicalNote
     from lxml import etree

     NS = "http://indivo.org/vocab/xml/documents#"

     class Transform(BaseTransform):

         def to_facts(self, doc_etree):
             args = self._get_data(doc_etree)

             # Create the fact and return it
             # Note: This method must return a list
             return [SimpleClinicalNote(**args)]

         def _tag(self, tagname):
             return "{%s}%s"%(NS, tagname)
         
         def _get_data(self, doc_etree):
             """ Parse the etree and return a dict of key-value pairs for object construction. """
             ret = {}
             _t = self._tag

             # Get the date_of_visit
             ret['date_of_visit'] = doc_etree.findtext(_t('dateOfVisit'))

             # Get the date finalized
             ret['finalized_at'] = doc_etree.findtext(_t('finalizedAt'))
        
             # Get the visit_type
             visit_type_node = doc_etree.find(_t('visitType'))
             ret['visit_type'] = visit_type_node.text
             ret['visit_type_type'] = visit_type_node.get('type')
             ret['visit_type_value'] = visit_type_node.get('value')
             ret['visit_type_abbrev'] = visit_type_node.get('abbrev')

             # Get the visit location
             ret['visit_location'] = doc_etree.findtext(_t('visitLocation'))

             # Get the specialty of the clinician
             specialty_node = doc_etree.find(_t('specialty'))
             ret['specialty'] = specialty_node.text
             ret['specialty_type'] = specialty_node.get('type')
             ret['specialty_value'] = specialty_node.get('value')
             ret['specialty_abbrev'] = specialty_node.get('abbrev')
             
             # get the signature
             signature_node = doc_etree.find(_t('signature'))
             provider_node = signature_node.find(_t('provider'))
             ret['signed_at'] = signature_node.findtext(_t('at'))
             ret['provider_name'] = provider_node.findtext(_t('name'))
             ret['provider_institution'] = provider_node.findtext(_t('institution'))

             # get the chief complaint
             ret['chief_complaint'] = doc_etree.findtext(_t('chiefComplaint'))
             
             # get the content of the note
             ret['content'] = doc_etree.findtext(_t('content'))
        
             return ret

.. automethod:: indivo.document_processing.BaseTransform.to_sdmj

   As an example, here's how you might implement this for a transform from our 
   :doc:`Simple Clinical Note Schema <schemas/scn-schema>` to our :doc:`Simple Clinical Note Data Model <data-models/scn>`. 
   Note the reuse of the ``_get_data()`` function from above::

     from indivo.document_processing import BaseTransform

     SDMJ_TEMPLATE = '''
     {
         "__modelname__": "SimpleClinicalNote",
         "date_of_visit": "%(date_of_visit)s",
         "finalized_at": "%(finalized_at)s",
         "visit_type": "%(visit_type)s",
         "visit_type_type": "%(visit_type_type)s",
         "visit_type_value": "%(visit_type_value)s",
         "visit_type_abbrev": "%(visit_type_abbrev)s",
         "visit_location": "%(visit_location)s",
         "specialty": "%(specialty)s",
         "specialty_type": "%(specialty_type)s",
         "specialty_value": "%(specialty_value)s",
         "specialty_abbrev": "%(specialty_abbrev)s",
         "signed_at": "%(signed_at)s",
         "provider_name": "%(provider_name)s",
         "provider_institution": "%(provider_institution)s
         "chief_complaint": "%(chief_complaint)s",
         "content": "%(content)s"
     }
     '''

     class Transform(BaseTransform):

         def to_sdmj(self, doc_etree):
             args = self._get_data(doc_etree)
             return SDMJ_TEMPLATE%args

.. automethod:: indivo.document_processing.BaseTransform.to_sdmx

   As an example, here's how you might implement this for a transform from our 
   :doc:`Simple Clinical Note Schema <schemas/scn-schema>` to our :doc:`Simple Clinical Note Data Model <data-models/scn>`. 
   Note the reuse of the ``_get_data()`` function from above::

     from indivo.document_processing import BaseTransform
     from lxml import etree
     from StringIO import StringIO

     SDMX_TEMPLATE = '''
     <Models>
       <Model name="SimpleClinicalNote">
         <Field name="date_of_visit">%(date_of_visit)s</Field>
         <Field name="finalized_at">%(finalized_at)s</Field>
         <Field name="visit_type">%(visit_type)s</Field>
         <Field name="visit_type_type">%(visit_type_type)s</Field>
         <Field name="visit_type_value">%(visit_type_value)s</Field>
         <Field name="visit_type_abbrev">%(visit_type_abbrev)s</Field>
         <Field name="visit_location">%(visit_location)s</Field>
         <Field name="specialty">%(specialty)s</Field>
         <Field name="specialty_type">%(specialty_type)s</Field>
         <Field name="specialty_value">%(specialty_value)s</Field>
         <Field name="specialty_abbrev">%(specialty_abbrev)s</Field>
         <Field name="signed_at">%(signed_at)s</Field>
         <Field name="provider_name">%(provider_name)s</Field>
         <Field name="provider_institution">%(provider_institution)s</Field>
         <Field name="chief_complaint">%(chief_complaint)s</Field>
         <Field name="content">%(content)s</Field>
       </Model>
     </Models>
     '''

     class Transform(BaseTransform):

         def to_sdmx(self, doc_etree):
             args = self._get_data(doc_etree)
             return etree.parse(StringIO(SDMX_TEMPLATE%args))

.. _add-transform:

Adding Custom Transforms to Indivo
----------------------------------

Associating a new transform with an Indivo-supported schema is simple: 

* Write your transform, as either an XSLT or a Python module, as described above.

* Drop the file containing your transform (``transform.xslt`` or ``transform.py``: make sure to name the file 
  'transform') into the directory containing the schema. See :ref:`add-schema` for more details. 

* Make sure to restart Indivo after moving transform files around, or the changes won't take effect.
