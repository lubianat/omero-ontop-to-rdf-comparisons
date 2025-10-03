
(WORK IN PROGRESS)

# Quick summary of core differences

This is a summary based on the long README documents, with the goal of getting some action items.

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

In other words, for the ontop mappings, the key become part of the property (If annotation keys are *controlled*, this modelling might be better. If they are free, though, minting URIs on-the-fly leads to unresolvable IDs)


# Next steps

## Standardize URIs for OMERO-RDF and OMERO-ONTOP .

### Agree on base URIs to be used on both estrategies. 

#### Proposal: 

* Following [this pull request](https://github.com/ome/omero-marshal/pull/84), RDF triples coming from omero databases SHOULD use the namespaces:
    * ome: http://www.openmicroscopy.org/Schemas/OME/2016-06#
    * omero: http://www.openmicroscopy.org/Schemas/OMERO/2016-06#

For OMERO-ONTOP, this would mean replacing `core: https://ld.openmicroscopy.org/core/` for `http://www.openmicroscopy.org/Schemas/OME/2016-06#`. `core` is used for clases on OMERO-ONTOP. 
It would also mean replace `omekg: https://ld.openmicroscopy.org/omekg/` for `http://www.openmicroscopy.org/Schemas/OME/2016-06#`.

For OMERO-RDF, this would mean replacing `ome: http://www.openmicroscopy.org/rdf/2016-06/ome_core/` for `http://www.openmicroscopy.org/Schemas/OME/2016-06#`.


Other namespaces MAY still be used (e.g. `ome_instance:	https://example.omero.server/` and `omens: <http://www.openmicroscopy.org/ns/default/>`), where not conflicting with the two standardized namespaces. 

#### Notes and consequences

Implementation on OMERO-ONTOP is technically straightforward, needing only changes in the namespace of the `omero-ontop-mappings.obda` and `omero-ontop-mappings.ttl`. 

The `omero-ontop-mappings.ttl` and `ome_core.ttl` ontologies from OMERO-ONTOP, as well as [`ome-owl`](gitlab.com/openmicroscopy/incubator/ome-owl) would be *deprecated* as sources of namespaces. 

A new ontology-based rendering of the OME Model, combining elements of the three, should be created with `http://www.openmicroscopy.org/Schemas/OME/2016-06#` as the base URI. 

The `http://www.openmicroscopy.org/Schemas/OME/2016-06#` prefix MAY be used to mind URIs that are not in the OME Model, but needed for a RDF specification. 

The same base URI will be used both for properties and classes.

#### Steps:

* Get feedback for the proposal (1-2 weeks), refine.
* Decide 


## Standardize properties for general relations

### Proposal

Use `DCTERMS:hasPart` (as OMERO-RDF does) and `rdfs:label` and `rdfs:description` (as OMERO-ONTOP does)


## Standardize the modeling of annotations

## Make the standardized URIs resolvable



## Setup a Sparnatural query interface that works with any OMERO SPARQL endpoint 
e.g. https://tiago.bio.br/omero-sparnatural/

## Build towards a federated network of OMERO-RDF metadata

In line with https://mpievolbio-scicomp.pages.gwdg.de/blog/post/20250627_lod4n4bi/


## Make URIs for _instance_ objects on public servers resolvable 

e.g. https://idr.openmicroscopy.org/webclient/?show=project-1152 

https://idr.openmicroscopy.org/Project/1152 should resolve to https://idr.openmicroscopy.org/webclient/?show=project-1152. 

For IRI resolvability, the minted IRI for instances could be something like: 

omero:idr.openmicroscopy.org/Project/1152 

This would make IRIs big-ish, but under an OME-run namespace, so it can resolve to a page explaining e.g. "this is a Project with id 1152 on an omero server runing on idr.openmicroscopy.org" 

There is a related discussion on the image.sc forum: https://forum.image.sc/t/uri-to-access-data-in-omero-using-the-omero-api/92571 