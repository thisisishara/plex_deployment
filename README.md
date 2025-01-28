# Profit and Loss Extractor
## Frontend:
- NextJS 14
- Node/NPM
- Shadcn/ui
- Docker

## Backend:
- Python 3.11
- Poetry
- Sanic
- Docker

## Links
[Backend Repository](https://github.com/thisisishara/plex_backend)
[Frontend Repository](https://github.com/thisisishara/plex_frontend)
[Deployment Repository](https://github.com/thisisishara/plex_deployment)

## How to Deploy & Run
First clone this repo using the following command: 
```bash
git clone --recurse-submodules https://github.com/thisisishara/plex_deployment.git
```
This will clone the frontend and backend projects required for the deployment which have been added to this repo as git submodules.

Please make sure to create the `.env` file in the `backend` directory as specified in the `README.md` file in the backend directory.

### ⚠️ Important
- Make sure the backend port is the same in both services in the docker compose file.
- Make sure to add a valid `mongodb connection URL` in the backend .env file. You can create a new Atlas cluster for free [here](https://www.mongodb.com/cloud/atlas/register). The connection string should look similar to `mongodb+srv://<username>:<password>@<cluster>.mongodb.net`.
- Make sure to add a valid `deepseek API Key` in the backend .env file. You can obtain after registering on [deepseek platform](https://platform.deepseek.com/sign_in). After registration is done, visit [here](https://platform.deepseek.com/api_keys).
- Without above configurations, the services will not work as expected.

After the .env files are configured within frontend and backend directories, you can simply run `docker compose up -d` to run the services using [Docker](https://docs.docker.com/engine/).

The frontend and backend directories also have dedicated `README.md` files explaining how to create and configure the .env files for each service.

## How to use the Frontend
### TL;DR - Docker
1. **Access**: Visit `http://localhost:3001` (or custom port).
2. **Upload**: Upload **one PDF at a time**; duplicates are detected.
3. **Select**: Choose a report and the **latest quarter** for P&L extraction.
4. **Extraction**: Toggle to extract only key P&L elements.
5. **Generate**: Click `Generate Profit and Loss Statement`, results displayed as a table and downloadable as CSV.
6. **Evaluate**: Click on `Evaluate`, Upload a reference CSV to calculate **precision, recall, and F1-score**.
7. **Features**:
   - Smart file handling.
   - Uses **Deepseek** and **LangChain** for extraction.
   - Compare results with reference CSV for performance.

### Explanation
Once the services are up and running, you can access the frontend using a browser. A newer browser is recommended for better UX.

Visit `http://localhost:3001` (replace the port if you configured a different port for the frontend) to load the `Financial Statement Analyzer` frontend app.

1. **Uploading Financial Statements**  
You can use the file upload input on the top of the app to select desired financial statements to be analyzed to extract Profit and Loass statements from them.
    - Only one file at a time
    - Only PDFs are allowed for now
    - Once the file is selected, click on `Upload` button to begin uploading.
2. **Select Report**  
After uploading, the file name should appear in the `Select Report` dropdown. The test financial statement files given for the coding challenge are already uploaded, so the dropdown should be already populated. The app is smart enough to detect the changes in already uploaded files. If you try to upload a file with the same name of an already uploaded file, the backend checks whether the content is identical by comparing file hashes and updates if needed. Else you'll be served with a nice toast saying the content is identical.  
When a new file is uploaded or content is changed, the backend uses [markitdown](https://github.com/microsoft/markitdown) to extract the markdown text from the provided PDF file and stores it in a mongodb collection with relevant metadata. Currently the backend does not perform any RAG operations of any sort (More on why in explained in the report submitted alongside this).

3. **Select Quarter**  
`Select Quarter` currently only has a single value since the task is supposed to retrieve the Profit and Loss statements for the `latest available quarter` in a given statement.

4. **Selected Item Extraction**  
You can also play around with `Enable Selected Item Extraction`, which forces the backend to only extract a pre-defined set of key elements related to Profit and Loass statements. This is done in the backend by altering the prompt instructions based on whether you have ticked the checkbox off or not. For brief extractions, it is recommended to keep this enabled.

5. **Generate the Statement** 
Finally you can click on `Generate Profit and Loss Statement` to start the extraction process.  
The backend loads the selected file content from mongodb and instantiates the ReportAnalyzer() which takes in the source file content, quarter, and other configurations and constructs a prompt based on a pre-defined langchain prompt template for the profit and loss extraction task. The prompt is passed to the deepseek via langchain's ChatOpenAI wrapper to perform a forced-tool call. (Langchain does not have a dedicated deepseek wrapper yet). The forced-tool call forces the model to always call a tool.   
The tool is a simple LangChain tool that has `extracted_items` argument which is a list of lists of elements. The idea is to extract the rows of the profit and loss statement as a list of lists of elements, each sub-list corresponds to a row. (Earlier I tried with requesting a CSV string and attempted to parse it manually, both OpenAI and Deepseek were unable to output valid CSV strings due to invalid symbols, which always resulted in invalid tool call arguments. List of lists approach was somewhat reliable.)  
This extracted list of rows are then passed back to the frontend to visualize it as a table (its easier to visualize since the results can be easily looped through and mapped to a table).   
    - Results are visualized under a new section in the bottom of the page as "Profit and Loss Statement"
    - It is also possible to download the result as a CSV (a frontend functionality, not backend)  

```python
@tool
def save_profit_and_loss_statement(
    extracted_items: Annotated[list[list[Any]], "Extracted profit and loss statement items for the requested quarter."],
) -> None:
    """Saves the extracted profit and loss statement CSV."""
    return None
```

6. **Evaluation**
It is also possible to calculate the `precision`, `recall` and `f1-score` for the extracted values. The reason for using those metrics was to get an overall idea on if the LLM has been able to extract **exact values** with high precision and recall. A metric like ROGUE or BLEU can be less effective here since they are more suitable for assessing lexical abilities of the model/approach.  
For calculate the metrics, you have to provide a valid CSV that has the same number of columns as the extracted Profit and Loss statement with the expected values specified for each line item. The row order does not matter, however the column order should be strickly maintained. The expected CSV also can contain additional line items the extracted CSV might not have (based on this, you can assess the recall of the approach.)  
The backend handles evaluation by converting the table (list of row lists) to a flattened dictionary with a unique keys based on `line item` of the row and the `column index`. For instance, this means, `profit before tax` of the 1st column (lets say Q3 of 2024) is always compared with the reference CSV's 1st column value under the `profit before tax` line item. The line items in the flatten dictionary looks like `profit before tax_0` where 0 represents the column index. So, as previously mentioned, when constructing the reference CSV file the column order should be identical to the extracted result.

## Backend Features
### Backend API
The backend Sanic server exposes the following routes:
- `/` (GET): Healthcheck
- `/api/v1/sources/` (GET): Retrieve all sources
- `/api/v1/sources/names` (GET): Retrieve all source names
- `/api/v1/sources/` (POST): Add a new source
- `/api/v1/sources/analyze` (POST): Analyze a specified source
- `/api/v1/results/evaluate` (POST): Evaluate an extracted P&L statement with a reference CSV

### Backend CLI
The backend also provides an easy to use `click` CLI, so if installed as a package in a python environment, you can quickly run the backend by executing `plex run` command. This is what is used in the Docker configs to run the backend service.


