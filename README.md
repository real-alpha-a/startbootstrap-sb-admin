from google.cloud import bigquery

def refresh_ingest_snapshot(project_id, dataset, report_name):
    client = bigquery.Client(project=project_id)

    query = f"""
    BEGIN TRANSACTION;

    DELETE FROM `{project_id}.{dataset}.ingest_table`
    WHERE report_name = @report_name
    AND report_year = (
        SELECT DISTINCT report_year
        FROM `{project_id}.{dataset}.land_table`
        WHERE report_name = @report_name
        ORDER BY meta_file_date DESC, bq_load_timestamp DESC
        LIMIT 1
    );

    INSERT INTO `{project_id}.{dataset}.ingest_table`
    SELECT *
    FROM `{project_id}.{dataset}.land_table`
    WHERE report_name = @report_name
    AND meta_file_date = (
        SELECT meta_file_date
        FROM `{project_id}.{dataset}.land_table`
        WHERE report_name = @report_name
        ORDER BY meta_file_date DESC, bq_load_timestamp DESC
        LIMIT 1
    );

    COMMIT TRANSACTION;
    """

    job_config = bigquery.QueryJobConfig(
        query_parameters=[
            bigquery.ScalarQueryParameter("report_name", "STRING", report_name)
        ]
    )

    query_job = client.query(query, job_config=job_config)
    query_job.result()  # Wait for completion

    print("Ingest table refreshed successfully.")
