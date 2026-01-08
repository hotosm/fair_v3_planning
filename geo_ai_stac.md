# STAC Standards In Geo-AI

Content copied from https://hackmd.io/@kshitijrajsharma/Sk4oxGqyee

Standards in Geo-AI: STAC-MLM, OGC TrainingDML-AI, and Related Initiatives
Introduction

Geospatial AI has rapidly grown, with applications ranging from land-cover mapping to disaster damage assessment. However, a historical lack of standards for organizing training data and models has led to fragmented practices (Yue 2023). In recent years, multiple initiatives have emerged to improve the FAIR (Findable, Accessible, Interoperable, Reusable) principles in GeoAI by standardizing how datasets and models are described and shared. This review examines key efforts, including the STAC-MLM extension for model metadata, the OGC TrainingDML-AI standard for training data, open-source frameworks like Raster Vision and TerraTorch, and commercial approaches such as Esri's GeoAI toolbox. We compare their scope, design, and adoption, and discuss trends and humanitarian use cases.
STAC-MLM: SpatioTemporal Asset Catalog – Machine Learning Model Extension

STAC-MLM is a community-driven extension to the SpatioTemporal Asset Catalog (STAC) specification, designed to catalog and describe machine learning models that use geospatial data (Wherobots 2023). Developed by organizations such as CRIM, Wherobots, Terradue, Radiant Earth, and NRCan, and presented at the 2024 ACM SIGSPATIAL conference (Charette-Migneault, Bédard, and Vincent 2024), STAC-MLM aims to make models discoverable alongside datasets and to capture all metadata needed for reuse or deployment. However, this is not the first one that tries to use STAC for machine learning it is more of successor of the extension which was started at Radiant Earth on October 4th, 2021. It was possibly the first STAC extension dedicated to describing machine learning models as per this PR

Overview of STAC-MLM
Overview of STAC-MLM

Key features of STAC-MLM include:

    Geospatial context: Spatial and temporal domain of the model, clarifying where and when it is applicable.
    Input data specifications: Required sensor bands or input types and any preprocessing steps (e.g., rescaling, normalization) needed (Wherobots 2023).
    Output description: The model's output shape, type, and semantic meaning (e.g., class labels for segmentation).
    Runtime requirements: An optional description of required runtime environments, frameworks, or hardware to ensure reproducibility.
    Provenance and references: Links to training details, scientific papers, or dataset sources.

STAC-MLM reuses STAC fields where possible and enables models to be represented as STAC Items or Collections, making them searchable via standard STAC APIs. By structuring model metadata consistently, it ensures that models become first-class, discoverable assets (Wherobots 2023).

The motivation behind STAC-MLM arose from inconsistent documentation practices, where data scientists repurposed general-purpose model cards without standardizing geospatial-specific metadata (Wherobots 2023). By introducing a common schema, STAC-MLM enables model portability and consistency, allowing a model published by one group to be understood and reused by others more easily.

Adoption: Despite being relatively new, STAC-MLM has seen early adoption. Wherobots' "Raster Inference" platform uses STAC-MLM for model imports, allowing users to bring their models with a JSON descriptor. Radiant Earth's MLHub and Terradue workflows have also integrated STAC-MLM, further demonstrating its practical utility (Wherobots 2023; Radiant Earth Foundation 2021). I particularly like this form builder for validating STAC for ML: ML Model stac validator. You can find schema reference here.

This is such a great initiative with so much potential, I envision a strong application as a having platform alike OpenAerialMap but for GeoAI Models.
OGC TrainingDML-AI: Standardizing Geospatial Training Data

While STAC-MLM focuses on model metadata, the OGC TrainingDML-AI standard addresses the organization of training datasets. Released by the Open Geospatial Consortium in 2023–2024, TrainingDML-AI defines a conceptual model and encoding methods (JSON, XML) for documenting geospatial ML training datasets (Open Geospatial Consortium 2023).

Key conceptual entities in TrainingDML-AI include:

    AI_TrainingDataset: The collection of training samples (e.g., a dataset for land cover classification).
    AI_TrainingData: A single training sample, such as an image and its corresponding labels.
    AI_Task: The ML task type (e.g., classification, detection, segmentation).
    AI_Label: Semantic description of labels, supporting scene-level, object-level, and pixel-level labels.
    AI_Labeling: Documentation of labeling procedures, provenance, and responsible entities, aligned with W3C PROV.
    DataQuality: Metadata describing dataset quality and uncertainties, following ISO 19157 standards.
    AI_TDChangeset: Versioning metadata to track updates or corrections between dataset releases.

TrainingDML-AI module overview
TrainingDML-AI module overview

TrainingDML-AI cleanly separates general metadata from domain-specific extensions, such as Earth Observation fields, supporting a wide range of AI domains while maintaining flexibility (Yue 2023).

The standard also defines JSON encoding formats to allow machine-readable and web-compatible descriptions, making it easier to publish, discover, and validate geospatial training datasets.

Comparison to STAC Label Extension: Before TrainingDML-AI, Radiant MLHub primarily used the STAC Label Extension to package training datasets. While effective, the STAC Label Extension focused primarily on imagery and label connections, lacking structured provenance, data quality, and versioning metadata. TrainingDML-AI expands this scope, offering a richer, more extensible model for dataset description (Yue 2023).
Radiant MLHub (Currently Source Cooperative) and Prior Community Efforts

Prior to the establishment of STAC-MLM and TrainingDML-AI, the geospatial community had already recognized the need for standardization around training datasets. Radiant Earth Foundation launched Radiant MLHub in 2019 as an open registry for geospatial ML datasets (Radiant Earth Foundation 2019). Radiant MLHub uses a STAC-compliant catalog and the STAC Label Extension to store imagery and corresponding labels in a consistent, machine-readable format. However I couldn't find the working link of radiant MLHUB at the moment. Only this Blog, As per this Article As of October 2023, all content previously available on Radiant MLHub has been migrated to Source Cooperative, and access to Radiant MLHub has been discontinued. Source Cooperative continues to be developed and supported by Radiant Earth, with the goal of making Earth science data more accessible and easier to use.

Each dataset on MLHub is organized as a STAC Collection of imagery Items and Label Items, with standardized metadata describing label classes, geospatial information, and dataset splits (train/validation). This early effort pioneered the FAIR publication of training datasets, allowing users to programmatically search and access resources through standard APIs (Radiant Earth Foundation 2021).

In addition to datasets, Radiant Earth expanded MLHub to include pretrained geospatial models by late 2021. These models were cataloged using an early version of the STAC ML Model Extension, effectively a precursor to STAC-MLM, linking models to their corresponding training datasets (Radiant Earth Foundation 2021). For example, Radiant Earth cataloged a tropical storm wind-speed estimation model alongside its training data, creating a clear lineage between data and models. This approach made it possible for researchers to query models based on task, region, or input data type, significantly improving model discoverability.

Other early community initiatives also contributed to standardization efforts. The OGC Testbed-18 project in 2022 examined metadata best practices for training datasets and recommended paths toward standardization (Open Geospatial Consortium 2023). Meanwhile, prototypes like the Deep Learning Metadata (DLM) proposal and an early ML Model STAC extension by groups such as Azavea and USGS laid important groundwork, although they lacked broad adoption. The design of the current STAC-MLM unified these earlier ideas into a more comprehensive and sustainable community standard.

In summary, early efforts like Radiant MLHub demonstrated that existing geospatial data standards (e.g., STAC) could be adapted to handle machine learning needs. These experiences informed the development of newer, formal standards like TrainingDML-AI and STAC-MLM, ensuring compatibility with the broader geospatial community's tools and practices.
Raster Vision: Open-Source GeoAI Pipeline

While standards like STAC-MLM and TrainingDML-AI address data and model documentation, open-source frameworks such as Raster Vision focus on operationalizing end-to-end machine learning pipelines for geospatial data. Initially developed by Azavea and now community-maintained, Raster Vision provides a configurable system for training and deploying models on overhead imagery (Azavea 2020).

Raster Vision Pipeline
Raster Vision Pipeline

Raster Vision supports common geospatial ML tasks, including:

    Chip Classification: Classifying small image patches (useful for land-cover classification).
    Object Detection: Detecting and localizing objects using bounding boxes.
    Semantic Segmentation: Per-pixel classification to delineate features like buildings, roads, and vegetation.

The framework handles geospatial-specific challenges such as very large image sizes, multiple spectral bands, and diverse coordinate systems. Raster Vision can ingest standard GIS data formats (e.g., GeoJSON, shapefiles) and satellite imagery (e.g., GeoTIFFs). It is flexible with input data formats, supporting labeling schemas like COCO JSON and Pascal VOC XML.

While Raster Vision does not impose a new data standard, it aligns well with community practices. For example, it can ingest imagery and labels organized using STAC catalogs. The developers have also expressed interest in deeper STAC integration, where data discovery could happen dynamically through STAC APIs (Azavea 2020).

Models trained with Raster Vision are bundled with configuration files and sample outputs, creating a semi-standardized model package. Although Raster Vision does not yet natively export STAC-MLM metadata, users could manually create an MLM JSON descriptor based on the training configuration, facilitating broader sharing and reuse.

In practice, Raster Vision has been widely adopted in the humanitarian and development sectors for tasks such as crop mapping in Africa and post-disaster building detection. Its emphasis on reproducible pipelines and flexible data handling makes it a key component in the emerging GeoAI ecosystem.
TerraTorch and Geospatial Foundation Models

The rise of geospatial foundation models; very large models pre-trained on broad Earth observation data has brought new tools and challenges to the GeoAI field. TerraTorch, developed by IBM Research and collaborators in 2023–2024, is an open-source toolkit designed to fine-tune and benchmark these foundation models (Kumar, Suresh, and Srivastava 2024).

Built on PyTorch Lightning, TerraTorch provides modular components specialized for satellite, weather, and climate data. It offers domain-specific data modules for handling multi-band imagery and time-series datasets, along with preconfigured tasks for classification, regression, and segmentation.

Key features of TerraTorch include:

    A model factory enabling users to swap pre-trained backbone encoders and task-specific decoder heads without coding.
    No-code fine-tuning workflows using YAML/JSON configurations.
    Automated hyperparameter tuning through an "Iterate" extension to optimize training runs.
    Integration with GEO-Bench, a benchmark suite offering standardized geospatial evaluation tasks for foundation models.

TerraTorch Pipeline
TerraTorch Pipeline

TerraTorch enables users to quickly adapt large pre-trained models, such as Prithvi-EO or Clay, to specific applications like flood mapping or crop classification. Although it does not define a new metadata standard, TerraTorch consumes data via standard geospatial loaders like TorchGeo, which can read STAC catalogs and related formats.

By integrating standardized evaluation suites like GEO-Bench, TerraTorch supports reproducible benchmarking of foundation models. As foundation models become more common in Earth observation, frameworks like TerraTorch will likely play a central role in operationalizing them for humanitarian and climate applications.
Commercial GeoAI Platforms: Esri's Approach

Commercial GIS platforms have also embraced GeoAI integration, with different approaches to standardization. Esri's ArcGIS Pro includes a GeoAI toolbox that provides tools for training and applying AI models on geospatial data (Esri 2024).

The GeoAI toolbox supports:

    Training regression and classification models on spatial tabular data.
    Object detection and pixel classification on imagery.
    Natural language processing tools for text geolocation and time-series forecasting.

Rather than creating new data standards, Esri ensures compatibility with widely used machine learning formats such as COCO, Pascal VOC, and KITTI. These formats allow data interoperability between ArcGIS and popular open-source frameworks like PyTorch and TensorFlow.

For model metadata, Esri defines the Esri Model Definition (.emd) format, a JSON structure describing input channels, class names, model architecture, and inference parameters. EMD files are bundled with model weights into Deep Learning Packages (.dlpk) for deployment within ArcGIS.

While EMD serves a similar purpose to STAC-MLM—capturing model metadata—it is Esri-specific and oriented toward its ecosystem. STAC-MLM remains platform-agnostic, intended for broader discovery and reuse across systems.

Esri's GeoAI tools emphasize user-friendliness, allowing non-programmers to train and deploy models within familiar GIS workflows. However, they also create some isolation: models and metadata created in ArcGIS may require conversion before being used in fully open ecosystems.
Comparison of Approaches

Different initiatives in GeoAI standardization target different parts of the machine learning lifecycle:

    Scope: STAC-MLM standardizes model metadata, while OGC TrainingDML-AI standardizes training data metadata. Raster Vision and TerraTorch focus on operationalizing workflows. Esri GeoAI offers end-to-end tools within its proprietary platform.
    Type: STAC-MLM and TrainingDML-AI are formal standards. Radiant MLHub and GEO-Bench implement standards. Raster Vision and TerraTorch are open-source frameworks. Esri GeoAI is a proprietary implementation.
    Community involvement: STAC-MLM and TrainingDML-AI were developed through open collaboration involving multiple stakeholders. Raster Vision and TerraTorch are maintained by open communities or open-source foundations. Esri's efforts are developed internally with user input but are closed source.
    Interoperability: STAC-MLM and TrainingDML-AI are designed for interoperability across platforms and organizations. Raster Vision and TerraTorch increasingly support standardized formats like STAC. Esri focuses on in-ecosystem workflows but allows for import/export in common formats.
    Metadata richness: TrainingDML-AI includes detailed provenance and data quality metadata. STAC-MLM captures model input requirements, output semantics, and runtime environments. Earlier practices often lacked this level of documentation.
    FAIR principles: All reviewed initiatives emphasize improving Findability, Accessibility, Interoperability, and Reusability, although with varying degrees of emphasis and maturity.

Overall, STAC-MLM and TrainingDML-AI complement each other, providing metadata coverage across the data-model pipeline. Raster Vision and TerraTorch operationalize GeoAI tasks, while Esri focuses on accessibility within a commercial GIS environment.
Trends in GeoAI Standardization

A major trend in the GeoAI landscape is the convergence on STAC as a backbone for organizing both geospatial datasets and machine learning models. Many initiatives, including Radiant MLHub, STAC-MLM, Raster Vision (roadmap), and AWS data catalogs, are either building on or planning support for STAC-based structures (Charette-Migneault, Bédard, and Vincent 2024; Radiant Earth Foundation 2021). This convergence enables more seamless discovery: an analyst could potentially query a single STAC API to find both input data and pretrained models for a given task.

Another trend is the formalization of community practices into industry standards. The involvement of the Open Geospatial Consortium (OGC) in formalizing TrainingDML-AI demonstrates that FAIR dataset practices are maturing into internationally recognized protocols. As TrainingDML-AI becomes more widely adopted, tools and repositories are likely to incorporate automated validation, metadata conversion, and training workflows based on these standards (Open Geospatial Consortium 2023).

Open-source tools are also evolving to integrate these standards. PySTAC libraries now support STAC-MLM, and emerging tools like pyTDML are aimed at supporting TrainingDML-AI datasets. Raster Vision and TerraTorch are aligning with open data access via STAC APIs and TorchGeo modules, respectively.

Finally, the rise of geospatial foundation models, such as IBM-NASA's Prithvi-EO and other self-supervised Earth observation models, is driving a need for standardized benchmarking and metadata. GEO-Bench provides a common evaluation suite, while TerraTorch operationalizes benchmarking workflows. This signals a broader maturing of GeoAI, moving from isolated experiments toward a reproducible, scalable discipline.

As GeoAI models move from research to production, deployment and containerization strategies have become critical for standardization. One prominent trend is using orchestration frameworks (Flyte, Kubeflow, ZenML, etc.) on Kubernetes clusters to run geospatial ML pipelines in a repeatable way. I need to do further research on this and perhaps include those topics in next blog.
Applications in Humanitarian GeoAI

Standardization efforts like STAC-MLM and TrainingDML-AI have significant implications for humanitarian applications of geospatial AI.

First, data sharing for disaster response becomes more effective. When multiple agencies contribute labeled datasets after an event, consistent metadata enables others to quickly find, validate, and reuse those datasets. TrainingDML-AI's support for provenance and quality metrics builds trust in shared resources, critical for decisions made under crisis conditions (Yue 2023).

Second, pretrained models for humanitarian tasks become more portable. Models for flood detection, crop failure prediction, or building damage assessment can be described using STAC-MLM metadata, making it easier for responders to identify and deploy relevant models without building new ones from scratch (Wherobots 2023).

Third, collaboration and capacity building benefit greatly from common standards. Volunteers, NGOs, and governments can work more easily together when datasets and models are described in interoperable ways. Standards lower technical barriers for cross-organizational efforts, increasing the reach of humanitarian AI initiatives. Which can be also seen in AI initiative called fAIr developed by Humanitarian OpenStreetMap Team (HOTOSM)

Fourth, transparency and ethics are improved. Standardized model and dataset descriptions enable users to assess the applicability and limitations of AI systems, a critical concern in humanitarian contexts where the consequences of model errors can be severe.

Finally, early examples such as Radiant Earth's MLHub datasets and SpaceNet challenges show that open, standardized datasets accelerate innovation and improve humanitarian outcomes by enabling broader reuse and benchmarking (Radiant Earth Foundation 2019).
Conclusion

The GeoAI field is undergoing a vital transformation through the adoption of standards such as STAC-MLM and OGC TrainingDML-AI. These initiatives address longstanding gaps in how geospatial datasets and machine learning models are documented, discovered, and reused. Open-source frameworks like Raster Vision and TerraTorch, and even commercial platforms like Esri's GeoAI toolbox, are increasingly integrating standard practices, signaling a broader convergence across sectors. Initiative like fAIr by HOTOSM which is trying to reduce the complexity for wider community to use AI in disasters

Yet challenges remain. Although these standards provide robust frameworks, real-world adoption is still limited, especially in humanitarian contexts where ease of use is crucial. While developing models and datasets can remain a technical task, deploying and operationalizing them during emergencies must become significantly easier. The lack of intuitive, user-friendly systems that leverage these standards points to an urgent need for further research and tool development.

As humanitarian challenges grow more complex and urgent, standardized, discoverable, and easily deployable GeoAI models and datasets will become foundational to effective response efforts. Continued collaboration between open communities, standards bodies, researchers, and commercial providers will be essential to realizing the full potential of a FAIR, interoperable, and impactful GeoAI ecosystem.

I might have missed some of the developments in the field, those are mentioned here are ones that I could find, Feel free to suggest me the updates or other initiatives that you are aware of. Thanks for reading ! Cheers !
AI Use Disclaimer

This document was prepared with the assistance of AI-based tools, including open source LLM LLAMA and OpenAI's ChatGPT. AI tools were used for structuring ideas, academic phrasing, and reference management based on user-provided research.
References

Azavea. 2020. "Raster Vision: Deep Learning for Aerial and Satellite Imagery." 2020. https://medium.com/pytorch/raster-vision-a-geospatial-deep-learning-framework-cd69ba840a83.

Charette-Migneault, François, Yvan Bédard, and Patrice Vincent. 2024. "Machine Learning Model Specification for Cataloging Spatio-Temporal Models." In Proceedings of the 32nd ACM SIGSPATIAL International Conference on Advances in Geographic Information Systems. https://doi.org/10.1145/3681769.3698586.

Esri. 2024. "GeoAI Toolbox in ArcGIS Pro." 2024. https://pro.arcgis.com/en/pro-app/latest/tool-reference/geoai/an-overview-of-the-geoai-toolbox.htm.

Kumar, R., A. Suresh, and M. Srivastava. 2024. "TerraTorch: Benchmarking and Fine-Tuning Geospatial Foundation Models." https://arxiv.org/abs/2503.20563.

Open Geospatial Consortium. 2023. "OGC Training Data Markup Language for Artificial Intelligence (TrainingDML-AI): Conceptual Model." OGC 23-008r3. Open Geospatial Consortium. https://docs.ogc.org/is/23-008r3/23-008r3.html.

Radiant Earth Foundation. 2019. "Radiant MLHub: An Open Registry for Geospatial Machine Learning." 2019. https://registry.opendata.aws/radiant-mlhub/.

———. 2021. "Geospatial Models in Radiant MLHub." 2021. https://medium.com/radiant-earth-insights/geospatial-models-now-available-in-radiant-mlhub-a41eb795d7d7.

Wherobots. 2023. "Raster Inference and STAC-MLM: Making Geospatial Models Discoverable." 2023. https://wherobots.com/raster-inference/.

Yue, P. 2023. "Toward Standardization of Machine Learning Training Datasets for Earth Observation." https://doi.org/10.1080/13658816.2022.2087223.
