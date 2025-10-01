Disclaimer: The .xsd file in this repository comes is licensed under  Copyright (C) 2002 - 2016 Open Microscopy Environment
(Creative Commons Attribution 3.0 Unported License.)


# Comparison of the OMERO-ONTOP and OMERO-RDF

OMERO-ONTOP and OMERO-RDF are two different projects towards a similar goal: making microscopy metadata from OMERO databases acessible from SPARQL endpoints. For that, the database information needs to be stored in RDF triples, and conversion is needed. 

To understand the details and differences of both, it is important to grasp some core concepts. They include:

* The OME Data Model, OME-XML and OME-TIFF 

* The representation of the OME Data Model in OMERO databases

* The ontologies and namespaces related to OME 

* The process of materialization of triples from a database (done by OMERO-RDF)

* The process of building a virtual knowledge graph on top of a relational database (done by OMERO-ONTOP)


Here I'll explore a bit the OME Data Model and its basis before diving into the intricacies of both approaches.
The overview will draw from multiple sources, including the list of OME publications at https://www.openmicroscopy.org/citing-ome/. 

## The OME Data Model, OME-XML and OME-TIFF 

While the vision for OME was shared in 2003, it was in [in 2005](https://genomebiology.biomedcentral.com/articles/10.1186/gb-2005-6-5-r47), that the OME Data Model" got first published on an article. It "defines a data model and a software implementation to serve as an informatics framework for imaging in biological microscopy experiments."

It does so organizes data around Projects, Datasets, Images and other elements. A good overall description is provided in the article itself:

_"All information in the OME Data Model can be reduced to 'semantic types' (STs). (...) STs can describe information at four levels in the OME hierarchy: Global, Dataset, Image and Feature. Global STs are used to describe 'Experimenters', 'Groups', 'Microscopes', and so on - items that are applicable to all images in an OME database. Dataset STs are used to describe information about datasets - information pertinent to a collection of images. Image STs describe information pertinent to images, and feature STs describe information about image features - objects or 'blobs' within images. In our nomenclature, the data type is an ST, and the data itself is an attribute. For example, the 'Pixels' data type is an Image ST, and a particular set of Pixels is an attribute of a particular Image."_

An XML representation of it was chosen, following XML's popularity at the time. The published 2005 article contains a "Why XML?" section, which includes the following: 

_"Modern browsers incorporate XML parsers, and are able to display the information contained in XML with the use of a style sheet (...). The use of XML also allows us to take advantage of its growing popularity in various unrelated fields - including a great deal of software written for XML, including databases, editing tools, and parsing libraries. Finally, and perhaps most important, XML is a plain-text format. As a last resort, it can be opened in any text editor and the information it contains can simply be read by a person. This inherent openness is one of its most desirable features for representing scientific data."_

The OME Data Model has evolved since 2005 and the latest version can be seen at https://docs.openmicroscopy.org/ome-model/latest/. 
As of September 2025, that is OME Data Model and File Formats 6.3.  An overview of the data model can be seen at https://ome-model.readthedocs.io/en/stable/developers/model-overview.html. 

Among the core pieces of software related to the model one may find the [BioFormats library](https://www.openmicroscopy.org/bio-formats/), which translates multiple kinds of metadata into the OME Model, and the [OMERO server for microscopy data](https://www.openmicroscopy.org/omero/), which uses the OME Model directly. 

The documentation outlines multiple recent changes to the data formats, but the latest change to the OME Model was around 2016. This can be seen at https://www.openmicroscopy.org/Schemas/ where one finds:

* OME - June 2016, version 2 [ome.xsd](https://www.openmicroscopy.org/Schemas/OME/2016-06/ome.xsd) The main schema which defines the OME ontology for microscopy, written in the [XML Schema Definition (.xsd)](https://en.wikipedia.org/wiki/XML_Schema_(W3C)) format

The model itself is described in full at https://www.openmicroscopy.org/Schemas/Documentation/Generated/OME-2016-06/ome.html. 

The current OME model clarifies a bit some changes from the original OME-XML format. The original OME-XML file encoded both the metadata AND the actual imaging data from the experiments. The 2005 article already presents why this might be counterintuitive: 

_"XML('s) human-readable design is often at odds with the storage of binary data. Since the bulk of an image file is represented by the pixels in the image and not the metadata, this might be perceived as a serious problem"

The problems was circumvented by using compression schemas, like gzip or bzip2, but the community wished for more performant variants. 
This was the point where TIFF entered the stage and the OME-TIFF specification appeared (https://ome-model.readthedocs.io/en/latest/ome-tiff/specification.html). By 2009, [an article by Jason Swedlow and colleagues](https://europepmc.org/backend/ptpmcrender.fcgi?accid=PMC3522875&blobtype=pdf) already mentioned that: 

_"While [the OME-XML approach is] conceptually sound, a more pragmatic approach is to store binary data as TIFF and then link image metadata represented as OME-XML by including it within the TIFF image header or as a separate file (37)."_

Of note reference 37 points to the currently broken URL http://www.loci.wisc.edu/ome/ome-tiff.html, which dated of 2008. 

As of 2025, as [https://github.com/ome/ome-model-documentation/issues/5 explained by Josh Moore] the _Pixels_ object in OME-XML file can "hold one of three types: 

* BinData: the original element of base64 encoded data. This was the "OME-XML as a file format" which has been superseded.

* TiffData: a block which explains how a given element in the XML maps to an offset within the TIFF

* MetadataOnly: such an OME-XML file has no direct link to the data it contains. However, this is used in OME-Zarr to embed OME metadata within the Zarr hierarchy. See https://ngff.openmicroscopy.org/0.5/index.html#bf2raw"

From the OME-Zarr 0.5 specification, it is clear that files adhering to the format:

* SHOULD have OME metadata representing the entire collection of images in a file named "OME/METADATA.ome.xml" which:
 * MUST adhere to the OME-XML specification but
 * MUST use <MetadataOnly/> elements as opposed to <BinData/>, <BinaryOnly/> or <TiffData/>;
 * MAY make use of the minimum specification.

This is a sort of digression from the original analysis of the OME Data Model itself, but very useful to understand how it plays out with the different formats. 

The focus of this comparison (OMERO-ONTOP vs OMERO-RDF) is agnostic with relation to the way the data itself is represented, but instead focuses on what the OME Data Model 2016 describes as the _Metadata Only Companion OME-XML file_

### Details of the 2016 OME-Model

After navigating the history of the OME Data Model, it is valuable to see some details of the XML Schema describing current, valid OME Models. 

First, it makes sure that instances of particular types have unique ids, e.g.: 

```xsd
    <xsd:key name="ImageIDKey"><xsd:selector xpath="OME:Image"/><xsd:field xpath="@ID"/></xsd:key>
    <xsd:key name="FolderIDKey"><xsd:selector xpath="OME:Folder"/><xsd:field xpath="@ID"/></xsd:key>
    <xsd:key name="WellIDKey"><xsd:selector xpath="OME:Plate/OME:Well"/><xsd:field xpath="@ID"/></xsd:key>
    <xsd:key name="LightSourceIDKey"><xsd:selector xpath="OME:Instrument/OME:Laser|OME:Instrument/OME:Arc|OME:Instrument/OME:Filament|OME:Instrument/OME:LightEmittingDiode|OME:Instrument/OME:GenericExcitationSource"/><xsd:field xpath="@ID"/></xsd:key>
```

So each Image, each Folder, each Well within a Plate and each LightSource (which may be of several types) get each one unique id. 

Then it makes sure references (like foreign keys) are clear: 

```
    <xsd:keyref name="FolderImageIDKeyRef" refer="OME:ImageIDKey" ><xsd:selector xpath="OME:Folder/OME:ImageRef"/><xsd:field xpath="@ID"/></xsd:keyref>
```

Here, each ImageRef (xpath="OME:Folder/OME:ImageRef") contained in a Folder must point to a valid identifier of an existing Image element, ensuring integrity across the model.

After setting the IDs and the references, the model continues on building the elements. I think it is worth it digging into one particular element as reference, say, <Image>. Comments that are particular to this document are preceded by "Local note:". 

```xsd
<!-- Key pixel storage elements -->
  <xsd:element name="Image"> <!-- top level definition -->
    <xsd:annotation>
      <xsd:appinfo><xsdfu xmlns=""><plural>Images</plural></xsdfu></xsd:appinfo>
      <xsd:documentation>
        This element describes the actual image and its meta-data.
         <!-- Local note: Plus other details. -->
      </xsd:documentation>
    </xsd:annotation>
    <xsd:complexType>
      <xsd:sequence>
     <!-- Local note: This is one "literal" typed bit of optional metadata  -->
        <xsd:element name="AcquisitionDate" minOccurs="0" maxOccurs="1" type="xsd:dateTime">
          <xsd:annotation>
            <xsd:documentation>
              The acquisition date of the Image.
              The element contains an xsd:dateTime string based on the ISO 8601 format (i.e. 1988-04-07T18:39:09.359)
             <!-- Local note: Plus other details. -->
            </xsd:documentation>
          </xsd:annotation>
        </xsd:element>

        <!-- Local note: Below there is one (optional) reference, linking an Image to a single Experimenter -->
        <xsd:element ref="ExperimenterRef" minOccurs="0" maxOccurs="1"/>

        <!-- Local note: Another "literal" typed bit of optional metadata, but now as a string  -->
        <xsd:element name="Description" minOccurs="0" maxOccurs="1">
          <xsd:annotation>
            <xsd:documentation>
              A description for the image. [plain-text multi-line string]
            </xsd:documentation>
          </xsd:annotation>
          <xsd:simpleType>
            <xsd:restriction base="xsd:string">
              <xsd:whiteSpace value="preserve"/>
            </xsd:restriction>
          </xsd:simpleType>
        </xsd:element>

        <!-- Local note: Other elements in this sequence have been ommited -->
        <!-- Local note: Now a reference to the Pixels object, which is mandatory-->
        <xsd:element ref="Pixels" minOccurs="1" maxOccurs="1"/>
  </xsd:element>
```

This is how the document looks like, overall, using standard .xsd syntax to describe what a valid OME-XML Metadata File looks like.

It goes on to specify elements like <Pixels>, <Experimenter>, <Project> and <Dataset>. 

A particularly interesting element (in terms of rendering the OME Model in RDF) is the one for Annotations, which hosts a set of different kinds of annotations compliant with the OME-Model. 

The <Image> specification already alludes to it: 
```xsd
    <xsd:element ref="AnnotationRef" minOccurs="0" maxOccurs="unbounded">
        <xsd:annotation>
        <xsd:appinfo><xsdfu xmlns=""><manytomany/></xsdfu></xsd:appinfo>
        </xsd:annotation>
    </xsd:element>
```
That is, annotations are optional and unbounded (there may be 0, 1 or many).There is also some relatively cryptic syntax like <xsdfu> and <manytomany/>, but probably not worth digging too much into that now. 
 
This ties neatly into the second part of the investigation: how the OME-XML metadata model is actually used in OMERO-Server PostgreSQL database. 

Before that, though, we may benefit from taking a look at the <StructuredAnnotations> element:


```xsd
 <xsd:element name="StructuredAnnotations"> <!-- top level definition -->
    <xsd:annotation>
      <xsd:documentation>
        An unordered collection of annotation attached to objects in the OME data model.
      </xsd:documentation>
    </xsd:annotation>
    <xsd:complexType>
      <xsd:choice minOccurs="0" maxOccurs="unbounded">
        <xsd:element ref="XMLAnnotation" minOccurs="1" maxOccurs="1"/>
        <xsd:element ref="FileAnnotation" minOccurs="1" maxOccurs="1"/>
        <xsd:element ref="ListAnnotation" minOccurs="1" maxOccurs="1"/>
        <xsd:element ref="LongAnnotation" minOccurs="1" maxOccurs="1"/>
        <xsd:element ref="DoubleAnnotation" minOccurs="1" maxOccurs="1"/>
        <xsd:element ref="CommentAnnotation" minOccurs="1" maxOccurs="1"/>
        <xsd:element ref="BooleanAnnotation" minOccurs="1" maxOccurs="1"/>
        <xsd:element ref="TimestampAnnotation" minOccurs="1" maxOccurs="1"/>
        <xsd:element ref="TagAnnotation" minOccurs="1" maxOccurs="1"/>
        <xsd:element ref="TermAnnotation" minOccurs="1" maxOccurs="1"/>
        <xsd:element ref="MapAnnotation" minOccurs="1" maxOccurs="1"/>
      </xsd:choice>
    </xsd:complexType>
  </xsd:element>


  ```

  So there are many kinds of Annotations that may appear, each with particular caracteristics and data structures. 

  Some of the examples include 

  ```xsd
  <xsd:element name="XMLAnnotation"> <!-- top level definition -->
    <xsd:annotation>
      <xsd:appinfo><xsdfu xmlns=""><plural>XMLAnnotations</plural></xsdfu></xsd:appinfo>
      <xsd:documentation>
        An general xml annotation. The contents of this is not processed as OME XML but should still be well-formed XML.
      </xsd:documentation>
    </xsd:annotation>
    <xsd:complexType>
      <xsd:complexContent>
        <xsd:extension base="TextAnnotation">
          <xsd:sequence>
            <xsd:element name="Value" minOccurs="1" maxOccurs="1">
              <xsd:complexType>
                <xsd:sequence>
                  <xsd:any processContents = "lax" minOccurs = "0" maxOccurs = "unbounded"/>
                </xsd:sequence>
              </xsd:complexType>
            </xsd:element>
          </xsd:sequence>
        </xsd:extension>
      </xsd:complexContent>
    </xsd:complexType>
  </xsd:element>
```

and 

```xsd
<xsd:element name="MapAnnotation"> <!-- top level definition -->
    <xsd:annotation>
      <xsd:appinfo><xsdfu xmlns=""><plural>MapAnnotations</plural></xsdfu></xsd:appinfo>
      <xsd:documentation>
        An map annotation. The contents of this is a list of key/value pairs.
      </xsd:documentation>
    </xsd:annotation>
    <xsd:complexType>
      <xsd:complexContent>
        <xsd:extension base="Annotation">
          <xsd:sequence>
            <xsd:element name="Value" type="Map" minOccurs="1" maxOccurs="1"/>
          </xsd:sequence>
        </xsd:extension>
      </xsd:complexContent>
    </xsd:complexType>
  </xsd:element>
```


## The representation of the OME Data Model in OMERO databases

Now that we have a basic understanding of the OME Data Model, let's see how it is used in practice by OMERO servers. 

It was [published in 2012](https://www.nature.com/articles/nmeth.1896) as _OME Remote Objects (OMERO), a software platform that enables access to and use of a wide range of biological data. OMERO uses a server-based middleware application to provide a unified interface for images, matrices and tables._ It follows on to describe that OMERO is not a single application, but many, or in their words,_"a tiered application of databases, middleware and remote client applications._

Figure 1B of the article describes the main information for our purposes here: _OME-XML is used by OMERO code generators to generate the OMERO relational database, the object-relational mapping model and the ICE-based remote-access system._

It is also relevant to understand the data flow, for example when importing files in proprietary formats to OMERO: 

_Data are imported into OMERO using Bio-Formats. All metadata are read and, where possible, mapped to the OME Data Model. Metadata are stored in the OMERO relational database, and binary pixel data are converted to an efficient binary-only file and stored in a file repository owned by the OMERO installation. To ensure data integrity **Bio-Formats also converts all proprietary file format metadata into a table of key-value pairs that are then stored as an annotation on the imported image in OMERO**. OMERO supports import using a desktop application (OMERO. importer), a command-line interface or a file-system monitoring tool (OMERO.Dropbox). Alternatively, third parties can write their own, specialized importers._

From there, it is possible to see that the Annotations in OMERO installations are indeed central, as they host anything that the OME Data Model was not able to capture beforehand. 

There are many databases core to OMERO, a "Binary Repository" flat file storage, hosting the binary data for the images, a Lucene Search Index and _the OMERO relational database, normally run by PostgreSQL (http://www.postgresql.org/), which holds all metadata associated with the binary images, all user information and most simple annotations, and records all write transactions in the OMERO installation._

The documentation page for the model on OMERO (https://omero.readthedocs.io/en/stable/developers/Model.html) clarifies how the OME Data Model relates to OMERO installations. It explains that

_Conceptually, the XSD files under the components/specification source directory are the starting point for all code generation. Currently however, the files in https://github.com/ome/omero-model under https://github.com/ome/omero-model/tree/v5.7.0/src/main/resources/mappings are hand-written based on the XSD files._

_The task created from the https://github.com/ome/omero-dsl-plugin/tree/v5.5.4/src Java files is then used to turn the mapping files into generated Java code in https://github.com/ome/omero-model under the build/classes/java/main directory. These classes are all within the ome.model package. A few hand-written Java classes can also be found in https://github.com/ome/omero-model/tree/v5.7.0/src/main/java/ome/model/internal._

Continuing that, one can see the use of Ice on how different clients may interact with the database:

_If we take a concrete example, a C++ client might create an Image via new omero::model::ImageI(). The “I” suffix represents an “implementation” in the Ice naming scheme and this subclasses from omero::model::Image. This can be remotely passed to the server which will be deserialized as an omero.model.ImageI object. This will then get converted to an ome.model.core.Image, which can finally be persisted to the database._

So Ice is a middleware that bridges client code and OMERO servers, allowing objects to be sent remotely via an internet connection.

The details of the implementation are rather complex — and perhaps unnecessary for the goal of comparing omero-ontop to omero-rdf. 

It may suffice to see that https://omero.readthedocs.io/en/stable/developers/Model/EveryObject.html# lists the objects available through OMERO, following the OME Data Model and that additional information are added via Annotations (e.g. see https://omero.readthedocs.io/en/stable/developers/Model/KeyValuePairs.html).These are accessed via the patterns described in the OMERO API (https://omero.readthedocs.io/en/stable/developers/Modules/Api.html).

In omero-rdf, for example, the model is acessed via omero-py using the BlitzGateway (https://omero.readthedocs.io/en/stable/developers/PythonBlitzGateway.html). For example:

```py
import omero
from omero.model import ProjectI
from omero.rtypes import rstring
p = ProjectI()
p.setName(rstring("Omero Model Project"))   # attributes are all rtypes
print(p.getName().getValue())               # getValue() to unwrap the rtype
print(p.name.val)                           # short-hand

from omero.gateway import ProjectWrapper
project = ProjectWrapper(obj=p)             # wrap the model.object
project.setName("Project Wrapper")          # Don't need to use rtypes
print(project.getName())
print(project.name)

print(project._obj)                  # access the wrapped object with ._obj
```

The gateway does some nice things, like hiding the use of `rstring` and other `rtypes` (the "remote types" abstraction that are needed for Ice to convert the OME Model between languages). For example, ProjectWrapper wraps the objects created with ProjectI(). A small note is that the "I" after Project (ProjectI) is an indication of the _client Implementation_ of a class that is read from the server. 

Also, documentation says that the Blitz gateway was originally built for the [OMERO.web framework](https://omero.readthedocs.io/en/stable/developers/Web.html) but has not yet been extended to wrap every omero.model object with a specific Blitz Object Wrapper. OMERO.web is a Django application, so it is built around Python bindings for OMERO servers. 

For example, in OMERO-RDF, the data is accessed like: 

```py
from omero.gateway import BlitzGateway, BlitzObjectWrapper
from omero.model import Dataset, Image, IObject, Plate, Project, Screen

# ... 

        elif isinstance(target, Dataset):
            # The _lookup gets the information about the dataset from OMERO
            ds = self._lookup(gateway, "Dataset", target.id)

            # The handler, then, processes the dataset information and gets an URI for the dataset in the RDF representation 
            dsid = handler(ds)
            ...
```

ONTOP, on the other hand, is a Java application that talks directly to the OMERO PostgreSQL instance via a JDBC driver. 

Now that we have a better idea of how the OME Data Model is actually embedded in OMERO, let's understand how it is materialized (or virtualized) in RDF triples. 

## The ontologies and namespaces related to OME 

Exposing data as RDF is considered a gold-standard FAIR practice. The Interoperability part of FAIR implies the use of common controlled vocabularies, so distrinct data resources can be integrated programatically. 

The standard practice is relying on reference ontologies, like the ones developed by the OBO Foundry consortium. 

There have been efforts in the past to have an explict OME Core Ontology written in the Web Ontology Language (OWL). There is small [2019 paper](https://ceur-ws.org/Vol-2849/paper-25.pdf) that says _the OME consortium has adopted the newly developed OWL-based OME model as an official companion to their XSD-based model._ It clarifies that _The OME core ontology is a translation of the OME-XML format of the OME data model version 2016-063 that covers all its concepts and attributes._ 


There is also a related 4DN-OME ontology, an extension of the OME-core ontology tied to an application called the Micro-Meta App, which _provides an interactive future-proof approach to document imaging experiments based on the 4DN-OME ontology and the proposed tiered-system of guidelines._ The app itself received a [description on Nature Methods](https://www.nature.com./articles/s41592-021-01315-z) in 2021, but it is somewhat orthogonal to OMERO-RDF. 


There is a [GitLab repository](https://gitlab.com/openmicroscopy/incubator/ome-owl) for ome-owl that is [mirrored on GitHub](https://github.com/German-BioImaging/ome-owl) by German BioImaging. The 4DN-OME ontology is hosted in the same GitHub repository as OME-OWL and imports it, e.g.: 

```owl

<http://www.4dnucleome.org/rdf/2019-05/> rdf:type owl:Ontology ;
    owl:versionIRI <http://www.4dnucleome.org/rdf/2019-05/> ;
    owl:imports <http://www.openmicroscopy.org/rdf/2016-06/ome_core/> .
```

Looking at the OME-core ontology, the following prefixes (among others) are defined:


```ttl
@prefix : <http://www.openmicroscopy.org/rdf/2016-06/ome_core/> .
@prefix mbmp: <http://www.openmicroscopy.org/rdf/2016-06/ome_core/MicrobeamManipuration#> .
@prefix expType: <http://www.openmicroscopy.org/rdf/2016-06/ome_core/ExperimentType#> .
@base <http://www.openmicroscopy.org/rdf/ome_core/> .
```

So it is a versioned ontology, with a base URL <http://www.openmicroscopy.org/rdf/ome_core/>  and a particular URL for this version <http://www.openmicroscopy.org/rdf/2016-06/ome_core/>, which ends up being the URI used in practice. 

Versioning ontologies [is a whole topic on its own](http://ontologydesignpatterns.org/index.php/Community:Versioning_and_URIs). In the OBO Foundry world, ontologies like Gene Ontology and the Experimental Factor Ontology deal with evolution in a different way:

* URIs are opaque and alpha-numeric, so they don't carry the weight of a language with them. Meaning, then, is attributed through labels, definitions and logical axioms.
* New ontology versions do not imply new URIs, as some are released very frequently. New URIs are minted for new concepts. 
* If a concept ends up not fitting the ontology anymore, it is deprecated. 

As the OME Core ontology uses directly English labels, readability for the URIs themselves is improved. There are a few details to note, though, as URIs then need to be more carefully minted. For example, the prefix declaration: 

```ttl
 @prefix mbmp: <http://www.openmicroscopy.org/rdf/2016-06/ome_core/MicrobeamManipuration#> .
```

Contains a typo which extends to the class definition itself: 

```ttl
###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/MicrobeamManipuration
:MicrobeamManipuration rdf:type owl:Class ;
                       rdfs:label "Microbeam Manipuration"@en ;
         rdfs:comment "Class \"MicrobeamManipuration\" is microbeam operation types and the regions of the images."@en .
```

The schema on ome_core.owl.ttl also defines

```ttl
###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/microbeamManipulation
:microbeamManipulation rdf:type owl:ObjectProperty ;
                       rdfs:domain :Experiment ,
                                   :Image ,
                                   :ImagingCondition ;
                       rdfs:range :MicrobeamManipuration ;
                       rdfs:label "microbeam manipuration"@en ;
                       rdfs:comment "Property \"microbeamManipuration\" specifies the microbeam operation that describes a microbeam operation type and the region of the image it was applied to."@en .
```

And there we find an even more confusing mix, including a camel-cased ":microbeamManipulation", with the capitalized "MicrobeamManipulation" in other places. Fixing the typo changes not the label, but the URI itself, potentially breaking downstream applications. 

As this ontology may be considered unstable and in development (the terms do not resolve, for example), this may be a minor issue. 

However, on moving towards a conciliation of OMERO-ONTOP and OMERO-RDF, it would be important for them to use a single, high-quality ontology as the source of IRIs. 

The OME-Core ontology is also relatively flat in terms of the class hierarchy. Most classes are only subclasses of `owl:Thing` and not connected to other ontologies, neither enriched with logical axioms. 

```ttl
###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/Image
:Image rdf:type owl:Class ;
       rdfs:label "Image"@en ;
         rdfs:comment "Class \"Image\" is actual images and their meta-data."@en .

```

 There are also several "individuals" in the ontology, e.g. 

 ```
 ###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/AcquisitionMode#brightField
<http://www.openmicroscopy.org/rdf/2016-06/ome_core/AcquisitionMode#brightField> rdf:type owl:NamedIndividual ,
                                                                                      :AcquisitionMode ;
                                                                             rdfs:label "bright field"@en .

###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/IlluminationType#epifluorescence
<http://www.openmicroscopy.org/rdf/2016-06/ome_core/IlluminationType#epifluorescence> rdf:type owl:NamedIndividual ,
                                                                                           :IlluminationType ;
                                                                                  rdfs:label "epifluorescence"@en .

###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/DetectorType#CMOS
<http://www.openmicroscopy.org/rdf/2016-06/ome_core/DetectorType#CMOS> rdf:type owl:NamedIndividual ,
                                                                              :DetectorType ;
                                                                     rdfs:label "CMOS"@en .
```
So in the ontology "brightField", "epifluorescence" and "CMOS" are `owl:NamedIndividual` 

This seem useful, but perhaps a bit semantically awkward, [as per OWL 2 specification](https://www.w3.org/TR/owl2-syntax/): 
for individuals. 

This is probably a consequence of using OWL to represent a data model. As [Chris Mungall put in his 2023 SWAT4HCLS Keynote](https://www.youtube.com/watch?v=2LRVrm8nVz4), one lesson learned is that "OWL is for modelling terminological knowledge, OWL is not a a language to describe data" and "OWL is a language for modelling a domain in an open-world fashion and when you try and use it in a closed-world fashion." He also points to a blogpost by someone that was using OWL as a _schema_ language called [Why I Don't Use OWL Anymore](https://www.topquadrant.com/resources/why-i-dont-use-owl-anymore/). He ends up moving towards talking about [LinkML](https://linkml.io/) and how it may fit some of the needs of our time.

In this case, the structure of the OME .owl ontology is perhaps less important than the URIs it mints. 

They are used downstream, e.g. in omero-rdf, as:

```json
{
            "@wd": "http://www.wikidata.org/prop/direct/",
            "@ome": "http://www.openmicroscopy.org/rdf/2016-06/ome_core/",
            "@ome-xml": "http://www.openmicroscopy.org/Schemas/OME/2016-06#",
            "@omero": "http://www.openmicroscopy.org/TBD/omero/",
            "@idr": "https://idr.openmicroscopy.org/",
        }
```

This use itself brings us another question. the "ome" and "ome-xml" prefixes are clear. What about the "omero" prefix, then, which includes a to be defined (TBD) as part of it? 

If I search github for the [URL](https://github.com/search?q=http%3A%2F%2Fwww.openmicroscopy.org%2FTBD%2Fomero%2F&type=code), there are a few instances of it being used, likely as a consequence of being baked in the omero-rdf pipeline. 
For example, there is a [set of triples using it](https://github.com/nfdi4plants/ARC-Symposium/blob/c3e5bf34a8d6fdc07e16ab464bf67cdba2af2c76/2024/RDF-Extension/1461.nt) from the [NFDI4PLANTS Hands-On Symposium on ARC Communication and Interoperability 2025](https://github.com/nfdi4plants/ARC-Symposium) and on [ERDERA/semantic-artifacts](https://github.com/ERDERA/semantic-artefacts), a project by Andra Waagmeester. 


There is a [draft pull request by Josh](https://github.com/ome/omero-marshal/pull/84) that tries and replaces the TBD in a top-level context: 

```py
{'@context': {'@base': 'http://www.openmicroscopy.org/Schemas/OME/2016-06#',
              'omero': 'http://www.openmicroscopy.org/Schemas/OMERO/2016-06#'},
```

This _omero_ prefix is aligned with the `.../Schemas/OME/2016-06#` namespace instead of the `.../rdf/2016-06/ome_core/` namespace. As of September 2025, another [draft PR](https://github.com/German-BioImaging/omero-rdf/pull/30) mentions that _All formats should now properly use OME 2016-06 as the default namespace and OMERO 2016-06 as the omero: ns._, which changes OMERO-rdf to use the "Schemas" namespace:

```py
# NAMESPACE DEFINITIONS
+ NS_OME = NS("http://www.openmicroscopy.org/Schemas/OME/2016-06#")
+ NS_OMERO = NS("http://www.openmicroscopy.org/Schemas/OMERO/2016-06#")
```

It is also clear that the "OMERO" namespace is different from the OME namespace. The OMERO namespace is related to https://www.openmicroscopy.org/Schemas/OMERO/, for which the last version dates from 2011. 


At the same time, the omero-ontop-mappings use a set of different namespaces. First, it provides [ome_core.ttl](https://github.com/German-BioImaging/omero-ontop-mappings/blob/main/omero-ontop-mappings/ome_core.ttl), described as:


```ttl
<https://ld.openmicroscopy.org/core/> rdf:type owl:Ontology ;
                                       dcterms:license "https://creativecommons.org/publicdomain/zero/1.0/" ;
                                       rdfs:label "OME-Core-Model" ;
                                       skos:definition "Linked data description of the OME Data Model" .
```

and it defines multiple classes according to the OME Model, such as 

```ttl
###  https://ld.openmicroscopy.org/core/Image
core:Image rdf:type owl:Class ;
           rdfs:subClassOf [ rdf:type owl:Restriction ;
                             owl:onProperty core:acquisition_date ;
                             owl:allValuesFrom linkml:Datetime
                           ] ,
                           [ rdf:type owl:Restriction ;
                             owl:onProperty core:description ;
                             owl:allValuesFrom linkml:String
                           ]
```

a very similar IRI, but now with lower case endings, denote some of the properties used, e.g.: 

```
###  https://ld.openmicroscopy.org/core/image
core:image rdf:type owl:ObjectProperty .
```

These properties don't have a description, though. It also models the XML Schema Descriptor enumerations as owl:NamedIndividuals, for example:

```
#################################################################
#    Individuals
#################################################################

###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/AcquisitionMode#FSM
<http://www.openmicroscopy.org/rdf/2016-06/ome_core/AcquisitionMode#FSM> rdf:type owl:NamedIndividual ,
                                                                              :AcquisitionMode ;
                                                                     rdfs:label "FSM"@en .


###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/AcquisitionMode#LCM
<http://www.openmicroscopy.org/rdf/2016-06/ome_core/AcquisitionMode#LCM> rdf:type owl:NamedIndividual ,
                                                                              :AcquisitionMode ;
                                                                     rdfs:label "LCM"@en .
```

A difference between the new, ONTOP-related ome_core.ttl and the other, older, .owl version is the use the LinkML model to specify types.

In particular, it uses the classes: 

```
linkml:permissible_values

linkml:ClassDefinition
linkml:SlotDefinition
linkml:EnumDefinition

linkml:Boolean
linkml:Datetime
linkml:Integer
linkml:String
```

### Detector types in .xsd and the OME ontologies

Let's see how the information DetectorType --> CMOS is encoded in the xsd, 2019 OME OWL and the ONTOP LD, with some detail omitted.  


```xsd
  <xsd:element name="Detector"> <!-- top level definition -->
    <xsd:complexType>
      <xsd:complexContent>
        <xsd:extension base="ManufacturerSpec">
          <xsd:attribute name="Type" use="optional">
            <xsd:annotation>
              <xsd:documentation>
                The Type of detector. E.g. CCD, PMT, EMCCD etc.
              </xsd:documentation>
            </xsd:annotation>
            <xsd:simpleType>
              <xsd:restriction base="xsd:string">
                <xsd:enumeration value="CCD"/><!-- Charge-Coupled Device -->
                <xsd:enumeration value="PMT"/><!-- Photomultiplier tube -->
                <xsd:enumeration value="CMOS"/><!-- complementary metal oxide semiconductor -->
              </xsd:restriction>
            </xsd:simpleType>
          </xsd:attribute>
        </xsd:extension>
      </xsd:complexContent>
    </xsd:complexType>
  </xsd:element>
```

OME-OWL 2019 aggregates "Detector" and "Type" in a single class DetectorType, but does not expecify a clear link between "Detector" and "DetectorType". It also does not restrict the valid DetectorTypes. 

"CMOS" is a owl:NamedIndividual, and an instance of the :DetectorType class.

```ttl
###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/Detector
:Detector rdf:type owl:Class ;
                 rdfs:label "Detector"@en ;
         rdfs:comment "Class \"Detector\" is types of detectors used to capture the image."@en .


###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/DetectorType#CMOS
<http://www.openmicroscopy.org/rdf/2016-06/ome_core/DetectorType#CMOS> rdf:type owl:NamedIndividual ,
                                                                              :DetectorType ;
                                                                     rdfs:label "CMOS"@en .
              
###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/DetectorType
        :DetectorType rdf:type owl:Class ;
                 rdfs:label "DetectorType"@en ;
         rdfs:comment "Class \"DetectorType\" is the types of detectors. E.g. CCD, PMT, EMCCD etc."@en .

```

Now the OMERO-ONTOP ontology has a few more steps. It also creates a DetectorType class, but CMOS is defined twice, first as an instance of owl:Class and then as an instance of owl:NamedIndividual. It is also, at the same time, an instance and a subclass of core:DetectorType.

This is an example of _punning_ in practice. Punning is the use of the same URI for multiple things, including a class and an individual, a feature [introduced in OWL 2](https://www.w3.org/TR/owl2-new-features/#F12%3a_Punning). It enables modelling of metaclasses. As OWL is does not allow real [https://nemo.inf.ufes.br/en/projetos/mlt/ multi-level modeling], the semantics becomes a bit hazy. Punning is a kind of a hack to insert multi-level modelling in OWL.

So "CMOS" is an owl:NamedIndividual of the metaclass core:DetectorType, while it is also a class of, say, the CMOS detector in a particular microscope. 

It uses two different ways to specify the valid detector types, one OWL-style, using a `owl:equivalentClass` , which closes the definition, and the other using [linkml:permissible_values](https://w3id.org/linkml/permissible_values), which defines _A list of possible values for a slot range._

So this .ttl doubles down as an schema and an ontology. 


```ttl
###  https://ld.openmicroscopy.org/core/DetectorType#CMOS
<https://ld.openmicroscopy.org/core/DetectorType#CMOS> rdf:type owl:Class ;
                                                       rdfs:subClassOf core:DetectorType .


###  https://ld.openmicroscopy.org/core/DetectorType#CMOS
<https://ld.openmicroscopy.org/core/DetectorType#CMOS> rdf:type owl:NamedIndividual ,
                                                                core:DetectorType .

<https://ld.openmicroscopy.org/core/DetectorType#CMOS> rdfs:label "CMOS" .

###  https://ld.openmicroscopy.org/core/Detector
core:Detector rdf:type owl:Class ;
              rdfs:subClassOf core:ManufacturerSpec ,
                              [ rdf:type owl:Restriction ;
                                owl:onProperty core:type ;
                                owl:allValuesFrom core:DetectorType
                              ] .

###  https://ld.openmicroscopy.org/core/DetectorType
core:DetectorType rdf:type owl:Class ;
                  owl:equivalentClass [ rdf:type owl:Class ;
                                        owl:unionOf ( <https://ld.openmicroscopy.org/core/DetectorType#APD>
                                                      <https://ld.openmicroscopy.org/core/DetectorType#CCD>
                                                      <https://ld.openmicroscopy.org/core/DetectorType#CMOS>
                            
                                                    )
                                      ] .

core:DetectorType linkml:permissible_values <https://ld.openmicroscopy.org/core/DetectorType#APD> ,
                                            <https://ld.openmicroscopy.org/core/DetectorType#CCD> ,
                                            <https://ld.openmicroscopy.org/core/DetectorType#CMOS>.
```


So, the same information may be modelled in different ways, but they should be specifying roughly the same things.

### Summary of the namespaces 

For the OMERO-RDF vs OMERO-ONTOP comparison, it is important to note the different namespaces there. 

The OME Data Model in XML makes use of [XML namespaces](https://en.wikipedia.org/wiki/XML_namespace).

```
xmlns:OME="http://www.openmicroscopy.org/Schemas/OME/2016-06"
```

That namespace is now the default by OMERO-RDF (with the addition of "OMERO=http://www.openmicroscopy.org/Schemas/OMERO/2016-06) in a recent draft [draft PR](https://github.com/German-BioImaging/omero-rdf/pull/30). 

Though, before, OMERO-RDF used  `<http://www.openmicroscopy.org/rdf/2016-06/ome_core/>` and `<http://www.openmicroscopy.org/TBD/omero/>`, the namespaces defined by the short-lived "OME-OWL" project, so legacy triples generated with the pipeline might use this.

The [ome-ld](https://github.com/joshmoore/ome-ld/blob/main/src/linkml.template) repository uses a different new prefix for representing data:

```
  core: https://ld.openmicroscopy.org/core/
```
That is the same that the OMERO-ONTOP MAPPINGs OME ontology/schema system uses. For the database mappings, some other namespaces are there, namely: 


```
omekg:		https://ld.openmicroscopy.org/omekg/
ome_instance:	https://example.org/site/
this:		https://ld.openmicroscopy.org/omekg#
```

The omekg namespace is defined on the OMERO-ONTOP mappings ontology

```
<https://ld.openmicroscopy.org/omekg> rdf:type owl:Ontology ;
                                       owl:imports rdfs: ,
                                                   core: ;
                                       dcterms:license "https://creativecommons.org/publicdomain/zero/1.0/" ;
                                       rdfs:label "OMERO VKG Ontology" ;
                                       skos:definition "Mapping ontology for OMERO Virtual Knowledge Graph" .
```

It uses constructs like:

```
@prefix core: <https://ld.openmicroscopy.org/core/> .

###  https://ld.openmicroscopy.org/omekg/Image
omekg:Image rdf:type owl:Class ;
            rdfs:isDefinedBy core:Image .

###  https://ld.openmicroscopy.org/core/Image
core:Image owl:equivalentClass omekg:Image .

```

So, in theory, omekg:Image is [defined by](https://www.w3.org/TR/rdf12-schema/#ch_isdefinedby) core:Image, while also being [equivalent](https://www.w3.org/TR/owl-ref/#equivalentClass-def) to it.


in some places in the OMERO-ONTOP mappings, one can find also the `omens` prefixs. It is used to mint URIs on the fly, as the base URL for keys in annotations with key-value pairs (e.g. creating `omens:{key}` on the fly).

```
prefix omens: <http://www.openmicroscopy.org/ns/default/>
```

Finally, there was a pull request for getting a w3id namespace (https://github.com/perma-id/w3id.org/pull/4579) which may yet move forward. 

<img src="https://imgs.xkcd.com/comics/standards.png"/>

Now that we have a better idea of the namespace zoo around the OME Model, we can dive deeper in OMERO-RDF vs the OMERO-ONTOP mappings.

## The process of materialization of triples from a database (done by OMERO-RDF)

As the poster ["There and back again"](https://zenodo.org/records/10687659) mentions, RDF was at the core of OMERO back in 2005, but it was descontinued due to performance issues. 

[OMERO-RDF](https://github.com/German-BioImaging/omero-rdf) is an OMERO plugin in Python that exports RDF from OMERO.


It takes an OMERO object and brings to life triples in a range of possible formats. These are specified in an OOP fashion. The code base in general goes around OOP design patterns, sometimes in a Java-like fashion, so a Java-esque mindset helps:  


```py
class Format:
    """
    Output mechanisms split into two types: streaming and non-streaming.
    Critical methods include:

        - streaming:
            - serialize_triple: return a representation of the triple
        - non-streaming:
            - add: store a triple for later serialization
            - serialize_graph: return a representation of the graph

    See the subclasses for more information.
    """
#... 


class StreamingFormat(Format):
#...
    def add(self, triple):
        raise RuntimeError("adding not supported during streaming")

    def serialize_graph(self):
        raise RuntimeError("graph serialization not supported during streaming")

class NTriplesFormat(StreamingFormat):
#...
    def serialize_triple(self, triple):
        s, p, o = triple
        escaped = o.n3().encode("unicode_escape").decode("utf-8")
        return f"""{s.n3()}\t{p.n3()}\t{escaped} ."""

class NonStreamingFormat(Format):
#...
    def add(self, triple):
        self.graph.add(triple)

    def serialize_triple(self, triple):
        raise RuntimeError("triple serialization not supported during streaming")
#...
```

These specify the formats, but the processing is done by a Handler class

```py


class Handler:
    """
    Instances are used to generate triples.
    """
#...
```

And it implements methods like

```py 
# Loading "handlers"
    def load_handlers(self) -> Handlers:
        annotation_handlers: Handlers = []
        eps = entry_points()
        for ep in eps.get("omero_rdf.annotation_handler", []):
            ah_loader = ep.load()
            annotation_handlers.append(ah_loader(self))
        return annotation_handlers

# Getting the identity of particular objects
    def get_identity(self, _type: str, _id: Any) -> URIRef:

        # Implementations of classes via ICE / RPC sometimes get an "I"
        # Say, Project becomes ProjectI
        # Small workaround for ROIs, though
        if _type.endswith("I") and _type != ("ROI"):
            _type = _type[0:-1]
        return URIRef(f"https://{self.info.host}/{_type}/{_id}")
```

Now, this is interesting already, because we have a new URI on the block. 
In here, the URI is defined by {self.info.host}, where info is a loaded server from `self.gateway.c.getRouter(comm).ice_getEndpoints()[0].getInfo()`.

If I am not missing something, that seems to be the URI for the OMERO server, which will be the base URI for the instances on that server. 

Now let's continue with other interesting methods. 

```py
    def get_bnode(self) -> BNode:
#...
            return BNode()
#...
    def get_type(self, data: Data) -> str:
        return data.get("@type", "UNKNOWN").split("#")[-1]


    def literal(self, v: Any) -> Literal:
        """
        Prepare Python objects for use as literals
        """
#...
        return Literal(v)

    def get_class(self, o):
        if isinstance(o, IObject):
            c = o.__class__
        else:  # Wrapper
            c = o._obj.__class__
        return c
```

That is interesting. So, what is an IObject in the OMERO Model?

After some web search, I got to: *IObject  the base interface for persistent model objects (things stored in the database). Classes like Image, Dataset, or Project implement IObject.* This line is *convenience so the Handler can work whether it’s handed a raw OMERO object or the Blitz wrapper.*

The Handle instances are also callable to get the data in the objects:

```py
    def __call__(self, o: BlitzObjectWrapper) -> URIRef:
#...  
            return self.handle(data)

```

Annotations in the objects are looped through individually. This is part of how OMERO-RDF handles annotations, which is may be  different from OMERO-ONTOP.

```py 

    def annotations(self, obj, objid):
        """
        Loop through all annotations and handle them individually.
        """
#... 
            for annotation in obj.listAnnotations(None):
                obj._loadAnnotationLinks() # Lazy loads annotation data from the server
                annid = self(annotation)
                self.contains(objid, annid) # Creates the triple with "hasParts" 
```

The self.contains() method is core to current OMERO-RDF functionality. It emits symmetric partonomy triples using <http://purl.org/dc/terms/>, as: 

```py
    def contains(self, parent, child):
        """
        Use emit to generate isPartOf and hasPart triples

        TODO: add an option to only choose one of the two directions.
        """
        self.emit((child, DCTERMS.isPartOf, parent))
        self.emit((parent, DCTERMS.hasPart, child))
```


than there is the handle method, inside the Handler class, that does the final emission of triples.
It uses here the formatters we mentioned before:

```py
    def emit(self, triple: Triple):
        if self.formatter.streaming:
            print(self.formatter.serialize_triple(triple), file=self.filehandle)
        else:
            self.formatter.add(triple)

    def handle(self, data: Data) -> URIRef:
        """
        Parses the data object into RDF triples.

        Returns the id for the data object itself
        """
#...
        for triple in self.rdf(_id, data):
            if triple:
                if None in triple:
                    logging.debug("skipping None value: %s %s %s", triple)
                else:
                    self.emit(triple)

        return _id
```

Sill in the methods, self.rdf() seems to be the method that actually turns data into triple objects before they are emitted. 

Given a particular subjects (_id), the method yields the triples as `Generator[Triple, None, None]`

```py

    def yield_object_with_id(self, _id, key, v):
        """
        Yields a link to the object as well as its representation.
        """
        v_type = self.get_type(v)
        val = self.get_identity(v_type, v["@id"])
        yield (_id, key, val)
        yield from self.rdf(_id, v)

    def rdf(self, _id: Subj, data: Data) -> Generator[Triple, None, None]:

        _type = self.get_type(data)

# The annotations are handled separately in a workaround 
        if "Annotation" in str(_type):
            for ah in self.annotation_handlers:
                handled = yield from ah(None, None, data,)
                if self.first_handler_wins and handled:
                    return
# ... 

       for k, v in sorted(data.items()):

            if k == "@type":
                yield (_id, RDF.type, URIRef(v))
  
            elif k in ("@id", "omero:details", "Annotations"):
                # Types that we want to omit for now
                pass

# Deciding between the OMERO and the OME URIs and cleaning the "omero:" prefix from the key
            else:
                if k.startswith("omero:"):
                    key = URIRef(f"{self.OMERO}{k[6:]}")
                else:
                    key = URIRef(f"{self.OME}{k}")

# If the value is an object...
                if isinstance(v, dict): 
# Values that have an @id get an URI 
                    if "@id" in v:
                        yield from self.yield_object_with_id(_id, key, v)

# Other values get a BNode
                    else:
                        bnode = self.get_bnode()
                        yield (_id, key, bnode)
                        yield from self.rdf(bnode, v)

# If the value is a list... 
                elif isinstance(v, list):

# Assumes multiple values, adds them with the original key 
                    for item in v:
                        if isinstance(item, dict) and "@id" in item:
                            yield from self.yield_object_with_id(_id, key, item)
                        elif isinstance(item, list) and len(item) == 2:
                            bnode = self.get_bnode()
# There is a big TODO here, but these are handled with he OME:Map, OME:Key and OME:Value properties
# The Key-Value pair gets in the RDF as literals, via OME:MAP to a BNode 
# 
# key:value then, becomes _id OME:Map [OME:Key "key"; OME:Value "value" . ] .  
                            yield (_id, URIRef(f"{self.OME}Map"), bnode)
                            yield (bnode, URIRef(f"{self.OME}Key"), self.literal(item[0]),)
                            yield (bnode, URIRef(f"{self.OME}Value"),  self.literal(item[1]),
                            )
                        else:
                            raise Exception(f"unknown list item: {item}")

# Finally, if it is not an object nor a list
                else:
                    yield (_id, key, self.literal(v))



        # Special handling for Annotations
        annotations = data.get("Annotations", [])
        for annotation in annotations:

            handled = False
            for ah in self.annotation_handlers:

# Yields links from _id to the annotation via OME:annotation
                handled = yield from ah(_id, URIRef(f"{self.OME}annotation"), annotation)
                if handled:
                    break

# uses an "AnnotationTBD" as a default 

            if not handled:  # TODO: could move to a default handler
                aid = self.get_identity("AnnotationTBD", annotation["@id"])
                yield (_id, URIRef(f"{self.OME}annotation"), aid)
                yield from self.rdf(aid, annotation)


```

So it goes through different kinds of options, generating triples accordingly. 

Annotations are handled differently, twice. First, if annotations are passed directly (say, a CommentAnnotation object) and in the second time, leaping through the annotations linked to the object there. 


Now, finally, there is the RdfControl class, the one that stitches the bits together. It gets some byts from the [omero-py cli](https://github.com/ome/omero-py/blob/master/src/omero/cli.py) module.

Before we get to that, it is useful to see it inherits the class BaseControl: 

```py
class BaseControl(object):
    """Controls get registered with a CLI instance on loadplugins().

    To create a new control, subclass BaseControl, implement _configure,
    and end your module with::

        try:
            register("name", MyControl, HELP)
        except:
            if __name__ == "__main__":
                cli = CLI()
                cli.register("name", MyControl, HELP)
                cli.invoke(sys.argv[1:])

    This module should be put in the omero.plugins package.

    All methods which do NOT begin with "_" are assumed to be accessible
    to CLI users.
    """
# ...
```

So that is the baseline on which RdfControl is built. Also imported are ProxyStringType() and Parser(ArgumentParser), but we won't go on that rabbit hole now. 


```py
class RdfControl(BaseControl):
    def _configure(self, parser: Parser) -> None:
        parser.add_login_arguments()
        rdf_type = ProxyStringType("Image")
        rdf_help = "Object to be exported to RDF"
        parser.add_argument("target", type=rdf_type, nargs="+", help=rdf_help)
# ...

# arguments: 
# --format (from list),
# --descent (recursive or flat)
# --ellide (boolean; shortens strings)
# --first-handler-wins (boolean; avoid duplicate annotations)
# --trim-whitespace (boolean; trim literals)
# --file (the target rdf file)
        parser.set_defaults(func=self.action)

# BlitzGateway decorator
    @gateway_required
    def action(self, args: Namespace) -> None:

        with open_with_default(args.file) as fh:
            handler = Handler(
                self.gateway,
# ...
                filehandle=fh,
            )
            self.descend(self.gateway, args.target, handler)
            handler.close()
# ...

# Then it descends, outputing URIRefs along the way 
    def descend(self, gateway: BlitzGateway, target: IObject, handler: Handler,) -> URIRef:
        """
        Copied from omero-cli-render. Should be moved upstream
        """

# Recursively enter lists and run descend
        if isinstance(target, list):
            return [self.descend(gateway, t, handler) for t in target]

# Skips if parsing is set to "flat"
        if handler.skip_descent():
            objid = handler(target)
            logging.debug("skip descent: %s", objid)
            return objid
        else:
# Increases the "self._descent_level" as it enters the nested hierarchy
            handler.descending()

```

After this basic setup, RDFControl, via the descend method, enters on processing the particular entries on the OME Model.

It follows a general pattern: 

* First it checks if a target is an instance of a particular class [Screen, Plate, Project, Dataset, Image]
* Then, it looks up the object and *handles* it. The handler does two things:
  * More clearly, it returns an ID for the object
  * But also, it goes through the object and *emits* the related triples

* Then it lists the children for the object, if it has nested children:
  * Screen -> Plate -> Well -> Image 
  * Project -> Dataset -> Image
  * Image -> Pixels 
  * Image -> ROI -> Shape

Nested levels are related with "contains" methods, or, in RDF-land, DCTERMS:hasPart

Additionally, every object can contain (DCTERMS:hasPart) annotations

Here is an example: 


```py
        if isinstance(target, Screen):
            scr = self._lookup(gateway, "Screen", target.id)
            scrid = handler(scr)
            for plate in scr.listChildren():
                pltid = self.descend(gateway, plate._obj, handler)
                handler.contains(scrid, pltid)
            handler.annotations(scr, scrid)
            return scrid
```

Let's look at the handler call: 


```py 
    def __call__(self, o: BlitzObjectWrapper) -> URIRef:
        c = self.get_class(o)
        encoder = get_encoder(c)
        if encoder is None:
            raise Exception(f"unknown: {c}")
        else:
            data = encoder.encode(o)
            return self.handle(data)
```

So, the data passed to self.handle is actually encoded from the object via this "encoder.encode(o)" function. 

That commes via [omero-marshal](https://github.com/ome/omero-marshal), which provides *Extensible marshaling code to transform various OMERO objects into dictionaries which can then be marshalled using JSON or alternative encodings.*

### Brief omero-marshall rabbit hole

It seems that a baseline understanding of omero-marshall might be needed to understand the OMERO-RDF triple parsing.
So, omero-marshall encoders inherit from a base "Encoder" class that looks like: 

```py
class Encoder(object):

    TYPE = ''

    def __init__(self, ctx):
        self.ctx = ctx

    def set_if_not_none(self, v, key, value):
        if value is None:
            return

# Checks if the value is a remote type via ICE or a base unit
        if isinstance(value, RType):
            v[key] = value.getValue()
        elif isinstance(value, UnitBase):
            self.encode_unit(v, key, value)
        else:
            v[key] = value

# Units get a TBD# namespace with the runtime class + name
    def encode_unit(self, v, key, value):
        v[key] = {
            '@type': 'TBD#%s' % value.__class__.__name__,
            'Unit': value.getUnit().name,
            'Symbol': value.getSymbol(),
            'Value': value.getValue()
        }

    def encode(self, obj):
# Objects get the TYPE set as a class variable; these are specified in the child classes
    def encode(self, obj):
        v = {'@type': self.TYPE}
        if hasattr(obj, 'id'):
            obj_id = unwrap(obj.id)
            if obj_id is not None:
                v['@id'] = obj_id
        if hasattr(obj, 'details') and obj.details is not None:
            encoder = self.ctx.get_encoder(obj.details.__class__)
            v['omero:details'] = encoder.encode(obj.details)

        return v
```

The encode method "unwraps" the remote types into Python-native structures, a big dict with a JSON-LD feel to it.
`@type` indicates the class, `@id` the object id and `omero:details` the extra details. 

Coming back to OMERO-RDF, it gets the encoder for the particular class via: 

```py
  c = self.get_class(o)
  encoder = get_encoder(c)
```


For the Screen class, [it](https://github.com/ome/omero-marshal/blob/57934f0fd75d0a3091d06b5e943094a27c56f328/omero_marshal/encode/encoders/screen.py) looks something like:

```py
class Screen201501Encoder(AnnotatableEncoder):

# Legacy namespace for screens, it is overwritten below
    TYPE = 'http://www.openmicroscopy.org/Schemas/SPW/2015-01#Screen'

# It sets data in this format for the different bits of the object
    def encode(self, obj):
        v = super(Screen201501Encoder, self).encode(obj)
        self.set_if_not_none(v, 'Name', obj.name)
        self.set_if_not_none(v, 'Description', obj.description)
        self.set_if_not_none(v, 'ProtocolDescription', obj.protocolDescription)
        self.set_if_not_none(v, 'ProtocolIdentifier', obj.protocolIdentifier)
#...

# And then repeats it for the lower parts
        if obj.isPlateLinksLoaded() and obj.sizeOfPlateLinks() > 0:
            plates = list()
            for plate_link in obj.copyPlateLinks():
                plate = plate_link.child
                plate_encoder = self.ctx.get_encoder(plate.__class__)
                plates.append(
                    plate_encoder.encode(plate)
                )
            v['Plates'] = plates
        return v


# In the 2016, the namespace was updated (woof! one namespace less to worry)
class Screen201606Encoder(Screen201501Encoder):

    TYPE = 'http://www.openmicroscopy.org/Schemas/OME/2016-06#Screen'


```
### Back from the omero-marshall rabbit hole

Ok, we were at this code snippet below:

```py
    def __call__(self, o: BlitzObjectWrapper) -> URIRef:
        c = self.get_class(o)
        encoder = get_encoder(c)
        if encoder is None:
            raise Exception(f"unknown: {c}")
        else:
            data = encoder.encode(o)
            return self.handle(data)
```

After getting the encoder from omero-marshal and encoding the data in a clearer format, it returns `self.handle(data)`.

It is the handle function we mentionded before, which runs the generator `self.rdf(_id, data)` and processes the omero-marshal data into RDF.

So, the handler returns the id and emits all the triples, generally following the omero-marshal model. 

The id is used, them, to emit the `hasParts` triples 


```py
        elif isinstance(target, Plate):
#... Do similar things for Plate
            return pltid

        elif isinstance(target, Project):
#... Project
            return prjid

        elif isinstance(target, Dataset):
#... Dataset
            return dsid

        elif isinstance(target, Image):
#... and Image, with a detail on the _get_rois method (see below)
            for roi in self._get_rois(gateway, img):
                roiid = handler(roi)
                handler.annotations(roi, roiid)
                handler.contains(pixid, roiid)
                for shape in roi.iterateShapes():
                    shapeid = handler(shape)
                    handler.annotations(shape, shapeid)
                    handler.contains(roiid, shapeid)
            return imgid

        else:
            self.ctx.die(111, "unknown target: %s" % target.__class__.__name__)
```

The only exception to the use of omero-marshall is the _get_rois method, which is implemented *de novo*: 

```py

    def _get_rois(self, gateway, img):
        params = ParametersI()
        params.addId(img.id)
        # Separated the query syntax for highlighting
        query = 
```
```sql

        select r from Roi r
                left outer join fetch r.annotationLinks as ral
                left outer join fetch ral.child as rann
                left outer join fetch r.shapes as s
                left outer join fetch s.annotationLinks as sal
                left outer join fetch sal.child as sann
                     where r.image.id = :id
```
```py
        return gateway.getQueryService().findAllByQuery(
            query, params, {"omero.group": str(img.details.group.id.val)}
        )
```

Okay. Now we have a good idea of the OMERO-RDF strategy for taking an OMERO database and converting it into RDF. 

We should now take a look at the OMERO-ONTOP-mappings and understand the strategy there better.

## The process of building a virtual knowledge graph on top of a relational database (done by OMERO-ONTOP)


The OMERO-ONTOP-MAPPINGS repo turns an OMERO database into RDF through a different strategy. It does not create RDF files directly but, instead, use ONTOP to create a virtual knowledge graph. There is also a [blogpost](mpievolbio-scicomp.pages.gwdg.de/blog/post/20250627_lod4n4bi/) explaining generally why this is being done on the first place. 

Driving now to the [ONTOP tutorial](https://ontop-vkg.org/tutorial/), it becomes clear that, besides a source database, there are three files needed for the plumbing: 


* An ontology file (OWL, e.g. .ttl)
* A mappings file (.obda)
* A properties file (.properties), which configures the JDBC drivers

This custom mapping language tells ONTOP the relations between the database and the ontology. 

Here is one example mapping from the OMERO-ONTOP repository: 

```
mappingId	MAPID-Image-0
target		ome_instance:Image/{image_id} this:acquisition_date {image_acquisitiondate}^^xsd:dateTime . 
source		select
			_image.id as image_id,
			_image.acquisitiondate as image_acquisitiondate
			from image as _image
```

Generally, identifiers become part of the URIs for objects in the database. This is similar to how OMERO-RDF deals with URIs; it is just important to make sure that the ome_instance prefix is the same as the host and that it uses "/" as the final separator.  

As a refresher, OMERO-RDF uses `return URIRef(f"https://{self.info.host}/{_type}/{_id}")`. 

For ONTOP, the way the IRI is constructed interferes directly with query performance or, in [their words](https://ontop-vkg.org/tutorial/mapping/uri-templates.html#manual-approach), *The choice of IRI templates may impact query complexity and performance, depending on whether joins have to be introduced to materialize those templates.* 

Let's go through the .obda file and understand the mapping strategy: 


```
[MappingDeclaration] @collection [[
mappingId	MAPID-classes
target		ome_instance:{Cls}/{id} a core:{Cls} . 
source		select id, concat(upper(left(cls,1)), right(cls, -1)) as Cls from (
			(select id,tableoid::regclass::text as cls, group_id from pixels)
			union
			(select id,tableoid::regclass::text as cls, group_id from image)
			union
# ...
			(select id,tableoid::regclass::text as cls, group_id from wellsample)
			) as object
			where group_id in (
			select parent as group_id from groupexperimentermap
			where child=0
			)
```

So, each entry gets an id (`ome_instance:{Cls}/{id}`) and a type via `rdf:type` (`core:{Cls}`).
The class itself comes from `tableoid::regclass::text`, which is basically the name of the table in PostgreSQL. 


```
target		
ome_instance:{Cls}/{id} a core:{Cls} ; 
                        dc:identifier {id}^^xsd:integer ; 
                        rdfs:label {name}^^xsd:string ;
                        rdfs:comment {description}^^xsd:string ;
                        core:owner ome_instance:Experimenter/{owner_id} ;
                        core:group ome_instance:ExperimenterGroup/{group_id} ;
                        this:update_id {update_id}^^xsd:integer ;
                        this:creation_id {creation_id}^^xsd:integer . 
                       
source		select id, ... from (
#...
			(select id,tableoid::regclass::text as cls, owner_id, group_id, update_id, creation_id, description, name from image)
			union
			(select id,tableoid::regclass::text as cls, owner_id, group_id, update_id, creation_id, description, name from project)
#...
			) as object
			where group_id in (
			select parent as group_id from groupexperimentermap
			where child=0
			)
```

Now we get a few more properties describing each entry. The local prefixes are: 

```
this:		https://ld.openmicroscopy.org/omekg#
core:		https://ld.openmicroscopy.org/core/
```

These relations are not explicit in omero-rdf, though. They come as consequences of the choices in the OME Data Model which bubble up into the RDF graph.

For example, instead of `rdfs:label`, omero-rdf uses `ome:Name`; 
instead of `rdfs:comment`, it uses `ome:Description`.

In a previous iteration of omero-rdf, there was a prototype for querying the graph, which relied on the output of omero-rdf: https://github.com/mpievolbio-scicomp/sparnatural-mpi/blob/main/mpi_graph/jointed-graphs.ttl. 

For example, in this previous iteration, the labels and descriptions comes via dedicated properties: 

```
@prefix ns1: <http://www.openmicroscopy.org/rdf/2016-06/ome_core/> .
@prefix ns2: <http://www.openmicroscopy.org/TBD/omero/> .

<https://ome.evolbio.mpg.de/Image/27789> a <http://www.openmicroscopy.org/Schemas/OME/2016-06#Image> ;
    ns2:series 0 ;
    ns1:Description "" ;
    ns1:Name "BER468_1Xylose_01_R3D_D3Dg.tif" .

<https://ome.evolbio.mpg.de/FileAnnotation/1980255> a <http://www.openmicroscopy.org/Schemas/OME/2016-06#FileAnnotation> ;
    ns1:Description "YAML file for bulk import of dataset 2873" ;
    ns1:File <https://ome.evolbio.mpg.de/OriginalFile/14197635> ;
    ns1:Namespace "/MouseCT/" .
```


The OMERO-ONTOP snipped before also adds `dc:identifier`, `core:owner`, `core:group`, `this:update_id` and `this:creation_id` which may not be represented in OMERO-RDF. 

Continuing to navigate the mappings, it is possible to see a different strategy to link Project --> Dataset: 

```
mappingId	MAPID-project-1
target		ome_instance:Project/{project_id} core:dataset ome_instance:Dataset/{dataset_id} . 
source		select
			_project.id as project_id,
			_projectdatasetlink.child as dataset_id
			from project as _project
			left join projectdatasetlink as _projectdatasetlink on _project.id = _projectdatasetlink.parent
```


So, a hard-coded "core:dataset" property is used, whereas in omero-rdf this was done by DCTERMS:hasPart.

Now, for annotations we find a slightly more intricate setup: 

target:

```ttl
ome_instance:Project/{project_id} <{map_key}> {map_value}^^xsd:string . 
```

source:
```sql
select
    _project.id as project_id,
    concat(
        regexp_replace(
            case
                when _annotation.ns is null
                    then 'http://www.openmicroscopy.org/ns/default/'
                when _annotation.ns ~ '^http[s]{0,1}:\/\/'
                    then _annotation.ns
                else 'http://www.openmicroscopy.org/ns/default/'
            end,
            '([^\/,#])$','\1/'
        ),
        regexp_replace(_annotation_mapvalue.name,'[^a-zA-Z0-9./_-]', 'ZZ', 'g')
    ) as map_key,

    _annotation_mapvalue.value as map_value

			from project as _project
			join projectannotationlink as _projectannotationlink on _projectannotationlink.parent = _project.id
			join annotation as _annotation on _projectannotationlink.child = _annotation.id
			join annotation_mapvalue as _annotation_mapvalue on _annotation.id = _annotation_mapvalue.annotation_id
```

While OMERO-RDF maps annotations using `ome:Key` and `ome:Key`, the OMERO-ONTOP mappings mint URIs for the keys on-the-fly. 

For that, it uses another namespace: `http://www.openmicroscopy.org/ns/default/` if it is a string (or not an URI) or whatever namespace the annotation key may already be using (`_annotation.ns`). 

This is a core difference between both workflows. The pattern is repeated for instances of `Dataset`, `Image`, `Well`, `Plate`, `Screen`, `Reagent` and `PlateAcquisition`, connecting these kind of objects to annotations

Now the .obda file enters the specification of triples about Images. 

### OMERO-ONTOP Images 
For example, it uses `this:acquisition_date` to extract the `xsd:dateTime` for acquisition, this is the `https://ld.openmicroscopy.org/omekg#` prefix:

```
mappingId	MAPID-Image-0
target		ome_instance:Image/{image_id} this:acquisition_date {image_acquisitiondate}^^xsd:dateTime . 
source		select
			_image.id as image_id,
			_image.acquisitiondate as image_acquisitiondate
			from image as _image
```

I could not figure out how this is information is represented (if at all) on omero-rdf, but it is there on the OMERO-ONTOP mappings. 

Here are some target statements from the .OBDA file that link to "ome_instance:Image{image_id}" The source SQL was omitted, as it can generally be understood here from context.

```
this:		https://ld.openmicroscopy.org/omekg#
core:		https://ld.openmicroscopy.org/core/

ome_instance:Image/{image_id} this:tag_annotation_value {tag_text}

ome_instance:Dataset/{dataset_id} core:image ome_instance:Image/{image_id} 

ome_instance:WellSample/{wellsample_id} core:image ome_instance:Image/{image_id} 

ome_instance:Pixels/{id} this:image ome_instance:Image/{image_id}

ome_instance:ROI/{roi_id} core:image ome_instance:Image/{image_id} 
```

So it seems that `this:tag_annotation_value` is used to represent tags, the `core:image` property links Datasets, WellSamples and ROIs to Images and `this:image` links Pixels to Image. 

The `this:tag_annotation_value` is also used on `Dataset`, `Project`, `Well`, etc.

### Other objects on OMERO-ONTOP 

```
ome_instance:ExperimenterGroup/{experimentergroup_id} 
            a :ExperimenterGroup ; 
            dc:identifier {experimentergroup_id}^^xsd:integer ;
            foaf:name {name}^^xsd:string .


ome_instance:Experiment/{experiment_id} 
            a :Experiment ; 
            dc:identifier {experiment_id}^^xsd:integer .

ome_instance:Experimenter/{id} a :Experimenter ; 
            dc:identifier {id}^^xsd:integer ; 
            foaf:firstName {firstname}^^xsd:string ; 
            foaf:lastName {lastname}^^xsd:string ; 
            foaf:email {email}^^xsd:string ; 
            foaf:name {name}^^xsd:string . 
source		select id, email, firstname, middlename, lastname, institution, concat("firstname", ' ', "lastname") as name from experimenter

ome_instance:Well/{well_id} core:plate ome_instance:Plate/{plate_id} . 

ome_instance:Channel/{id} 
            a :Channel ;
            dc:identifier {id}^^xsd:integer ;
            this:blue {blue}^^xsd:integer ;
            this:red {red}^^xsd:integer ;
            this:green {green}^^xsd:integer ;
            this:pixels ome_instance:Pixels/{pixels_id} . 

ome_instance:Pixels/{id} 
            a :Pixels ; 
            this:image ome_instance:Image/{image_id} ; 
            this:physical_size_x {physicalsizex}^^xsd:float ; 
            this:physical_size_x_unit {physicalsizexunit}^^xsd:string ; 
            this:physical_size_y {physicalsizey}^^xsd:float ; 
            this:physical_size_y_unit {physicalsizeyunit}^^xsd:string ; 
            this:physical_size_z {physicalsizez}^^xsd:float ; 
            this:physical_size_z_unit {physicalsizezunit}^^xsd:string . 
```

Oh, here we have something interesting again. 

On the sample OMERO-RDF, this is how a Pixels object looks like:

```
<https://ome.evolbio.mpg.de/Pixels/4684881> a <http://www.openmicroscopy.org/Schemas/OME/2016-06#Pixels> ;
    dcterms:isPartOf <https://ome.evolbio.mpg.de/Image/4684931> ;
    ns2:sha1 "49fbe3ecfbdcb83f8013e62dad3a038038c28560" ;
    ns1:PhysicalSizeX [ ] ;
    ns1:PhysicalSizeY [ ] ;
    ns1:SignificantBits 16 ;
    ns1:SizeC 1 ;
    ns1:SizeT 1 ;
    ns1:SizeX 1008 ;
    ns1:SizeY 672 ;
    ns1:SizeZ 1 ;
    ns1:Type <https://ome.evolbio.mpg.de/PixelsType/6> .
```
It is generated by a snippet that says 

```py
if img.getPrimaryPixels() is not None:
    pixid = handler(img.getPrimaryPixels())
    handler.contains(imgid, pixid)
```
So, there `PhysicalSizeX` points to a [ ] and the `PhysicalSizeXUnit` is not there.
The [omero-marshal encoder for Pixels](https://github.com/ome/omero-marshal/blob/master/omero_marshal/encode/encoders/pixels.py)

The [omero-gateway wrapper](https://github.com/ome/omero-py/blob/master/src/omero/gateway/__init__.py) seems to normalize the PhysicalSizeX (and Y and Z) in microns.

```py
    @assert_pixels
    def getPixelSizeX(self, units=None):
        """
        Gets the physical size X of pixels in microns.
        If units is True, or a valid length, e.g. "METER"
        return omero.model.LengthI.

        :return:    Size of pixel in x or None
        :rtype:     float or omero.model.LengthI
        """
        return self._unwrapunits(
            self._obj.getPrimaryPixels().getPhysicalSizeX(), units)
```

While the getPrimaryPixels() method: 

```
    @assert_pixels
    def getPrimaryPixels(self):
        """
        Loads pixels and returns object in a :class:`PixelsWrapper`
        """
        return PixelsWrapper(self._conn, self._obj.getPrimaryPixels())
```


Diving into this may be a bit beyond the scope here. 

This is basically what gets currently in as triples in the OMERO-ONTOP mappings schema. 

I think at this point we may move towards a summary of the differences.

# Quick summary of core differences

Throughout the bullet points, (1) refers to omero-rdf, (2) refers to omero-ontop

* Different namespaces for similar / identical things
  * (1) http://www.openmicroscopy.org/rdf/2016-06/ome_core/ (legacy)
  * (1) http://www.openmicroscopy.org/Schemas/OME/2016-06# (to be used in place of /rdf/)
  * (2) https://ld.openmicroscopy.org/core/

* Besides these namespaces, there are many other namespaces used here and there

* Nesting parts are organized with "DCTERMS:hasPart" on OMERO-RDF and custom properties on the ONTOP mappings, e.g.:
    * (1) Dataset --- DCTERMS:hasPart ---> Image 
    * (2) Dataset --- core:Image ---> Image

* Descriptions and labels are also handled with different properties, e.g.:

    * (1) Image --- ome:Name ---> "name.ome.zarr" ; Image --- ome:Description ---> "the description" 
    * (2) Image --- rdfs:label ---> "name.ome.zarr" ; Image --- rdfs:comment ---> "the description"

* The OMERO-ONTOP mappings use a few other custom properties to link, say, a Project to an Experimenter (`core:owner`)

* Key-Value annotations are handled in different ways:
    * (1) Object --- ome:annotation ---> Blank Node 
        * Blank Node --- ome:Value ---> "value"
        * Blank Node --- ome:Key ---> "key"
    * (2) Object ---> annotation_ns:"key" ---> "value"

In other words, for the ontop mappings, the key become part of the property 

    (If annotation keys are *controlled*, this modelling might be better. If they are free, though, minting URIs on-the-fly leads to unresolvable IDs)


### To figure out

* I am still not sure exactly how Pixel "PhysicalSizeX" and "PyisicalSizeXUnit" get in the omero-rdf model. 