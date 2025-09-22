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

(to be continued)


## The ontologies and namespaces related to OME 

(build upon https://ceur-ws.org/Vol-2849/paper-25.pdf)