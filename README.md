Disclaimer: Ontologies and schemas in this repository are provided under multiple different.

The .ttl (Turtle) ontology files come from https://github.com/German-BioImaging/ome-owl. 

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

As a personal note, while XML may not be everyone's favourite format in 2025, it was perhaps the most user-friendly format back in the days. 

The OME Data Model has evolved since 2005 and the latest version can be seen at https://docs.openmicroscopy.org/ome-model/latest/. 
As of September 2025, that is OME Data Model and File Formats 6.3.  An overview of the data model can be seen at https://ome-model.readthedocs.io/en/stable/developers/model-overview.html. 

Among the core pieces of software related to the model one may find the [BioFormats library](https://www.openmicroscopy.org/bio-formats/), which translates multiple kinds of metadata into the OME Model, and the [OMERO server for microscopy data](https://www.openmicroscopy.org/omero/), which uses the OME Model directly. 

The documentation outlines multiple recent changes to the data formats, but the latest change to the OME Model was around 2016. This can be seen at https://www.openmicroscopy.org/Schemas/ where one finds:

* OME - June 2016, version 2 [ome.xsd](https://www.openmicroscopy.org/Schemas/OME/2016-06/ome.xsd) The main schema which defines the OME ontology for microscopy, written in the [XML Schema Definition (.xsd)](https://en.wikipedia.org/wiki/XML_Schema_(W3C)) format

The model itself is described in full at https://www.openmicroscopy.org/Schemas/Documentation/Generated/OME-2016-06/ome.html. 

The current OME model clarifies a bit some changes from the original OME-XML format. The original OME-XML file encoded both the metadata AND the actual imaging data from the experiments. The 2005 article already presents why this might be counterintuitive: 

_"XML('s) human-readable design is often at odds with the storage of binary data. Since the bulk of an image file is represented by the pixels in the image and not the metadata, this might be perceived as a serious problem"

The problems was circumvented by using compression schemas, like gzip or bzip2, but the community wished for more performant variants. 
This was the point where TIFF entered the stage and the OME-TIFF specification appeared (https://ome-model.readthedocs.io/en/latest/ome-tiff/specification.html). By 2009, [https://europepmc.org/backend/ptpmcrender.fcgi?accid=PMC3522875&blobtype=pdf an article by Jason Swedlow and colleagues] already mentioned that: 

_"While [the OME-XML approach is] conceptually sound, a more pragmatic approach is to store binary data as TIFF and then link image metadata represented as OME-XML by including it within the TIFF image header or as a separate file (37)."_

Of note reference 37 points to the currently broken URL http://www.loci.wisc.edu/ome/ome-tiff.html, which dated of 2008. 

As of 2025, as [https://github.com/ome/ome-model-documentation/issues/5 explained by Josh Moore] the _Pixels_ object in OME-XML file can _"hold one of three types: 

* BinData: the original element of base64 encoded data. This was the "OME-XML as a file format" which has been superseded.

* TiffData: a block which explains how a given element in the XML maps to an offset within the TIFF

* MetadataOnly: such an OME-XML file has no direct link to the data it contains. However, this is used in OME-Zarr to embed OME metadata within the Zarr hierarchy. See https://ngff.openmicroscopy.org/0.5/index.html#bf2raw"_

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
That is, annotations are optional and unbounded (there may be 0, 1 or many).

There is also some relatively cryptic syntax like <xsdfu> and <manytomany/>, which ChatGPT 5 said are _not part of standard XML Schema but rather OMERO-specific extensions embedded inside <xsd:appinfo>. Standard XSD parsers ignore them, but OMERO’s code generation tools interpret them as ORM (object–relational mapping) hints. In this case, <manytomany/> marks the relationship between Image and Annotation as a many-to-many link, which in the relational database translates into a join table connecting images and annotations.
In other words, this snippet shows the dual nature of the OME-XML schema:
* Validation layer → ensures OME-XML metadata files are structurally valid.
* Metadata layer → provides machine-readable hints that allow the schema to be projected into other models (Java classes, database schemas, RDF graphs, etc.)._
 
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

It was [https://www.nature.com/articles/nmeth.1896 published in 2012] as _OME Remote Objects (OMERO), a software platform that enables access to and use of a wide range of biological data. OMERO uses a server-based middleware application to provide a unified interface for images, matrices and tables._ It follows on to describe that OMERO is not a single application, but many, or in their words,_"a tiered application of databases, middleware and remote client applications._

Figure 1B of the article describes the main information for our purposes here: _OME-XML is used by OMERO code generators to generate the OMERO relational database, the object-relational mapping model and the ICE-based remote-access system._

It is also relevant to understand the data flow, for example when importing files in proprietary formats to OMERO: 

_Data are imported into OMERO using Bio-Formats. All metadata are read and, where possible, mapped to the OME Data Model. Metadata are stored in the OMERO relational database, and binary pixel data are converted to an efficient binary-only file and stored in a file repository owned by the OMERO installation. To ensure data integrity **Bio-Formats also converts all proprietary file format metadata into a table of key-value pairs that are then stored as an annotation on the imported image in OMERO**. OMERO supports import using a desktop application (OMERO. importer), a command-line interface or a file-system monitoring tool (OMERO.Dropbox). Alternatively, third parties can write their own, specialized importers._

From there, it is possible to see that the Annotations in OMERO installations are indeed central, as they host anything that the OME Data Model was not able to capture beforehand. 

There are many databases core to OMERO, a "Binary Repository" flat file storage, hosting the binary data for the images, a Lucene Search Index and _the OMERO relational database, normally run by PostgreSQL (http://www. postgresql.org/), [which] holds all metadata associated with the binary images, all user information and most simple annotations, and records all write transactions in the OMERO installation._

The documentation page for the model on OMERO (https://omero.readthedocs.io/en/stable/developers/Model.html) clarifies how the OME Data Model relates to OMERO installations. It explains that

_Conceptually, the XSD files under the components/specification source directory are the starting point for all code generation. Currently however, the files in https://github.com/ome/omero-model under https://github.com/ome/omero-model/tree/v5.7.0/src/main/resources/mappings are hand-written based on the XSD files._

_The task created from the https://github.com/ome/omero-dsl-plugin/tree/v5.5.4/src Java files is then used to turn the mapping files into generated Java code in https://github.com/ome/omero-model under the build/classes/java/main directory. These classes are all within the ome.model package. A few hand-written Java classes can also be found in https://github.com/ome/omero-model/tree/v5.7.0/src/main/java/ome/model/internal._

Continuing that, one can see the use of Ice on how different clients may interact with the database:

_If we take a concrete example, a C++ client might create an Image via new omero::model::ImageI(). The “I” suffix represents an “implementation” in the Ice naming scheme and this subclasses from omero::model::Image. This can be remotely passed to the server which will be deserialized as an omero.model.ImageI object. This will then get converted to an ome.model.core.Image, which can finally be persisted to the database._

So Ice is a middleware that bridges clent code and an OMERO servers. I also asked GPT 5 to clarify a bit how it works:

_Under the hood, the objects are defined in Java and exposed remotely through the Ice middleware. Ice acts as the bridge between client code (such as omero-py) and the OMERO server, serializing requests such as getName() or saveAndReturnObject() into a binary protocol (including abstracted types) and routing them to the server. On the server side, these calls are deserialized back into method invocations on the corresponding Java model classes, which then persist data into PostgreSQL or retrieve it from the database. This setup ensures that clients written in Python, Java, C++, or MATLAB all interact with the same underlying object model without having to deal directly with SQL or the physical database schema._

The details of the implementation are rather complex — and perhaps unnecessary for the goal of comparing omero-ontop to omero-rdf. 

It may suffice to see that https://omero.readthedocs.io/en/stable/developers/Model/EveryObject.html# lists the objects available through OMERO, following the OME Data Model and that additional information are added via Annotations (e.g. see https://omero.readthedocs.io/en/stable/developers/Model/KeyValuePairs.html).These are accessed via the patterns described in the OMERO API (https://omero.readthedocs.io/en/stable/developers/Modules/Api.html). OMERO.py follows the patterns (https://github.com/ome/omero-py)

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


There is also a related 4DN-OME ontology, an _extension of the OME-core ontology, specifically tailored at enhancing the reproducibility and comparison of single-molecule, super-resolution fluorescence microscopy experiments._ It implements, among other things, an extension of _the existing OME core-classes `Instrument’ and `Image’ to reflect the technological advances and the quality control requirements associated with single-molecule, super-resolution microscopy._

It was tied to an application called the Micro-Meta App, which _provides an interactive future-proof approach to document imaging experiments based on the 4DN-OME ontology and the proposed tiered-system of guidelines._ The app itself received a [description on Nature Methods](https://www.nature.com./articles/s41592-021-01315-z) in 2021.


There is a [GitLab repository](https://gitlab.com/openmicroscopy/incubator/ome-owl) for ome-owl that is [mirrored on GitHub](https://github.com/German-BioImaging/ome-owl) by German BioImaging. The 4DN-OME ontology is hosted in the same GitHub repository as OME-OWL and imports it, e.g.: 

```owl

<http://www.4dnucleome.org/rdf/2019-05/> rdf:type owl:Ontology ;
    owl:versionIRI <http://www.4dnucleome.org/rdf/2019-05/> ;
    owl:imports <http://www.openmicroscopy.org/rdf/2016-06/ome_core/> .
```

Looking at the OME-core ontology, the following prefixes (among others) are defined:


```
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

Alan Ruttenberg mentions on a mailing list that _that doesn't mean that there shouldn't be some way to track the history of a term. However, I would have that happen by annotation, change notes, etc rather than creating a new URI to replace the old one._ I tend to agree with him, at least for _reference_ ontologies like the ones in OBO. OME-OWL is not a reference ontology, though, but more akin to an application ontology. 

The difference between reference and application ontologies are also a topic on its own. But it may be reasonable to say that the OME Core ontology is tied to the OMERO / OME Data Model and the representation of microscopy-related metadata, and not general considerations about the world. These differences may justify or base some of the design decisions of the ontology. 

As the OME Core ontology uses directly English labels, readability for the URIs themselves is improved. There are a few details to note, though, as URIs then need to be more carefully minted. For example, the prefix declaration: 

```
 @prefix mbmp: <http://www.openmicroscopy.org/rdf/2016-06/ome_core/MicrobeamManipuration#> .
```

Contains a typo which extends to the class definition itself: 

```
###  http://www.openmicroscopy.org/rdf/2016-06/ome_core/MicrobeamManipuration
:MicrobeamManipuration rdf:type owl:Class ;
                       rdfs:label "Microbeam Manipuration"@en ;
         rdfs:comment "Class \"MicrobeamManipuration\" is microbeam operation types and the regions of the images."@en .
```

The schema on ome_core.owl.ttl also defines

```
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

```
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

_5.6 Individuals_

_Individuals in the OWL 2 syntax represent actual objects from the domain.(...)_

_Named individuals are identified using an IRI. Since they are given an IRI, named individuals are entities.(...)_

_The individual a:Peter can be used to represent a particular person. It can be used in axioms such as the following one: `ClassAssertion( a:Person a:Peter )` 	Peter is a person._ 


This is probably a consequence of using OWL to represent a data model. As [Chris Mungall put in his 2023 SWAT4HCLS Keynote](https://www.youtube.com/watch?v=2LRVrm8nVz4):one lesson learned is that "OWL is for modelling terminological knowledge, OWL is not a a language to describe data" and "OWL is a language for modelling a domain in an open-world fashion and when you try and use it in a closed-world fashion." He also points to a blogpost by someone that was using OWL as a _schema_ language called [Why I Don't Use OWL Anymore](https://www.topquadrant.com/resources/why-i-dont-use-owl-anymore/). He ends up moving towards talking about [LinkML](https://linkml.io/) and how it may fit some of the needs of our time.



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

```
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

```
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

These properties don't have a description, though. 

It also models the XML Schema Descriptor enumerations as owl:NamedIndividuals, for example:


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

A difference between the new, ONTOP-related ome_core.ttl and the old .owl version is the use the LinkML model to specify types.
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

Let's see how the information DetectorType --> CMOS is encoded in the xsd, 2019 OME OWL and the ONTOP LD, with some detail ommitted.  


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

This is an example of _punning_ in practice. Punning is the use of the same URI for multiple things, including a class and an individual, a feature [introduced in OWL 2](https://www.w3.org/TR/owl2-new-features/#F12%3a_Punning). It enables modelling of metaclasses. As OWL is does not allow real [https://nemo.inf.ufes.br/en/projetos/mlt/ multi-level modeling], the semantics becomes a bit hazy. Punning is (IMO) a kidn of a hack to insert multi-level modelling in OWL.

So "CMOS" is an owl:NamedIndividual of the metaclass core:DetectorType, while it is also a class of, say, the CMOS detector in a particular microscope. 

core:DetectorType and core:Detector are also linked.


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

Though, before, OMERO-RDF used  `<http://www.openmicroscopy.org/rdf/2016-06/ome_core/>` and `<http://www.openmicroscopy.org/TBD/omero/>`, the namespaces defined by the short-lived "OME-OWL" project, so legacy triples generated with the pipeline might use this IRS

The [ome-ld](https://github.com/joshmoore/ome-ld/blob/main/src/linkml.template) repository uses a differentnew prefix for representing data in LinkML format:

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


in some places in the OMERO-ONTOP mappings, one can find also the `omens` prefixs

```
prefix omens: <http://www.openmicroscopy.org/ns/default/>
```

Finally, there was a pull request for getting a w3id namespace (https://github.com/perma-id/w3id.org/pull/4579) which may yet move forward. 

<img src="https://imgs.xkcd.com/comics/standards.png"/>

Now that we have a better idea of the namespace zoo around the OME Model, we can dive deeper in OMERO-RDF vs the OMERO-ONTOP mappings.

## The process of materialization of triples from a database (done by OMERO-RDF)

As the poster ["There and back again"](https://zenodo.org/records/10687659) mentions, RDF was at the core of OMERO back in 2005, but it was descontinued due to performance issues. 

[OMERO-RDF](https://github.com/German-BioImaging/omero-rdf) is an OMERO plugin in Python that exports RDF from OMERO.


It takes an OMERO object and brings to life triples in a range of possible formats. These are specified in an OOP fashion: 


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

class TurtleFormat(NonStreamingFormat):
#..
    def serialize_graph(self) -> None:
        return self.graph.serialize()

class JSONLDFormat(NonStreamingFormat):
#...
    def serialize_graph(self) -> None:
        return self.graph.serialize(
            format="json-ld",
            context=self.context(),
            indent=4,
        )


class ROCrateFormat(JSONLDFormat):
#...
    def serialize_graph(self):
        ctx = self.context()
        j = pyld_jsonld_from_rdflib_graph(self.graph)
#...
        return json.dumps(j, indent=4)
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

# Deciding between the OMERO and the OME URIs 
    def get_key(self, key: str) -> Optional[URIRef]:

        # Plus a few types that are omitted
        if key in ("@type", "@id", "omero:details", "Annotations"):
            # Types that we want to omit fo
            return None
        else:
            if key.startswith("omero:"):
                # clean the "omero:" prefix from the key
                return URIRef(f"{self.OMERO}{key[6:]}")
            else:
                return URIRef(f"{self.OME}{key}")

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




