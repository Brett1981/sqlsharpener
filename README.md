# SqlSharpener
Parses SQL files to create a meta-object hierarchy with which you can generate C# code either manually or by invoking the included pre-compiled T4 template. The included template will automatically generate C# stored procedure wrappers.

# Motivation
Rather than generating code from the database or using a heavy abstraction layer that might miss differences between the database and data access layer until run-time, this project aims to provide a very fast and simple data access layer that is generated at design-time using SQL _files_ as the source-of-truth (such as those found in an SSDT project).

# Example
**Given a stored procedure that looks like this:**

    CREATE PROCEDURE [dbo].[usp_TaskGet]
    	@TaskId int
    AS
  	-- Specifying "TOP 1" makes the generated return value a single instance instead of an array.
  	SELECT TOP 1
  		t.Name,
  		t.[Description],
  		ts.Name as [Status],
  		t.Created,
  		t.CreatedBy,
  		t.Updated,
  		t.UpdatedBy
  	FROM Tasks t
  	JOIN TaskStatus ts ON t.TaskStatusId = ts.Id
  	WHERE t.Id = @TaskId

**The included pre-compiled template will generate C# code like this:**

    // ------------------------------------------------------------------------------
    // <auto-generated>
    //     This code was generated by SqlSharpener.
    //  
    //     Changes to this file may cause incorrect behavior and will be lost if
    //     the code is regenerated.
    // </auto-generated>
    // ------------------------------------------------------------------------------
    namespace SimpleExample.DataLayer
    {
    	using System;
    	using System.IO;
    	using System.Data;
    	using System.Data.SqlClient;
    	using System.Configuration;
    	using System.Collections.Generic;

    	public interface IStoredProcedures
    	{
    		int usp_TaskCreate( String Name, String Description, Int32? TaskStatusId, DateTime? Created, String CreatedBy, DateTime? Updated, String UpdatedBy, out Int32? TaskId );
    		usp_TaskGetDto usp_TaskGet( Int32? TaskId );
    		int usp_TaskUpdate( Int32? TaskId, String Name, String Description, Int32? TaskStatusId, DateTime? Updated, String UpdatedBy );
    	}
    
    	public partial class StoredProcedures : IStoredProcedures
    	{
    		public usp_TaskGetDto usp_TaskGet( Int32? TaskId )
    		{
    			usp_TaskGetDto result = null;
    			var connectionString = ConfigurationManager.ConnectionStrings["ConnectionString1"].ConnectionString;
    			using(var conn = new SqlConnection(connectionString))
    			{
    				conn.Open();
    				using (var cmd = conn.CreateCommand())
    				{
    					cmd.CommandType = CommandType.StoredProcedure;
    					cmd.CommandText = "usp_TaskGet";
    					cmd.Parameters.Add("TaskId", SqlDbType.Int).Value = (object)TaskId ?? DBNull.Value;
    
    					using(var reader = cmd.ExecuteReader(CommandBehavior.SequentialAccess))
    					{
    						while (reader.Read())
    						{
    							var item = new usp_TaskGetDto();
    							item.Name = reader.GetString(0);
    							item.Description = reader.GetString(1);
    							item.Status = reader.GetString(2);
    							item.Created = reader.GetDateTime(3);
    							item.CreatedBy = reader.GetString(4);
    							item.Updated = reader.GetDateTime(5);
    							item.UpdatedBy = reader.GetString(6);
    							result = item;
    						}
    						reader.Close();
    					}
    
    				}
    				conn.Close();
    			}
    			return result;
    		}
      }
    
    	public class usp_TaskGetDto
    	{
    		public String Name { get; set; }
    		public String Description { get; set; }
    		public String Status { get; set; }
    		public DateTime? Created { get; set; }
    		public String CreatedBy { get; set; }
    		public DateTime? Updated { get; set; }
    		public String UpdatedBy { get; set; }
    	}
    
    
    }

# Installation

Using Nuget, run the following command to install SqlSharpener:

    PM> Install-Package SqlSharpener
    
This will add SqlSharpener as a solution-level package. That means that the dll's do not get added to any of your projects (nor should they). 

# Generation

Add a new T4 template (*.tt) file to your data project. Set its content as follows:

    <#@ template debug="false" hostspecific="true" language="C#" #>
    <#@ assembly name="$(SolutionDir)\packages\SqlSharpener.1.0.2\tools\SqlSharpener.dll" #>
    <#@ output extension=".cs" #>
    <#@ import namespace="Microsoft.VisualStudio.TextTemplating" #>
    <#@ import namespace="System.Collections.Generic" #>
    <#@ import namespace="SqlSharpener" #>
    <#
    	// Specify paths to your SQL files.
    	var sqlPaths = new List<string>();
    	sqlPaths.Add(Host.ResolvePath(@"..\SimpleExample.Database\dbo\Tables"));
    	sqlPaths.Add(Host.ResolvePath(@"..\SimpleExample.Database\dbo\Stored Procedures"));
    
    	// Set parameters for the template.
    	var session = new TextTemplatingSession();
    	session["outputNamespace"] = "SimpleExample.DataLayer";
    	session["connectionStringVariableName"] = "ConnectionString1";
    	session["procedurePrefix"] = "usp_";
    	session["sqlPaths"] = sqlPaths;
    
    	// Generate the code.
    	var t = new SqlSharpener.StoredProceduresTemplate();
        t.Session = session;
    	t.Initialize();
    	this.Write(t.TransformText());
    #>

Right-click on the .tt file and click "Run Custom Tool". The generated .cs file will contain a class with functions for all your stored procedures, DTO objects for procedures that return records, and an interface you can used if you use dependency-injection.

# Usage

Once the code is generated, your business layer can call it like any other function. Here is one example:

        public int? Create(Task task)
        {
            int? taskId;
            storedProcedures.TaskCreate(
                task.Name,
                task.Description,
                task.TaskStatusId,
                task.Created,
                task.CreatedBy,
                task.Updated,
                task.UpdatedBy,
                out taskId);
            return taskId;
        }
        
# Dependency Injection

If you use a dependency-injection framework such as Ninject, you can use the interface generated. For example:

    public class  DataModule : NinjectModule
    {
        public override void Load()
        {
            Bind<IStoredProcedures>().To<StoredProcedures>().InSingletonScope();
        }
    }
    
