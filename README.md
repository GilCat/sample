using System;
using System.Collections.Generic;
using System.Data.Services.Client;
using System.Linq;
using System.Net;
using System.Text.RegularExpressions;
using System.Threading;
using Microsoft.SystemCenter.Orchestrator.WebService;
using SMACommon.Encryption;



namespace SMAOrchestrator
{
    public class Program
    {
        public static String MethodName = "";
        public static int Status;
        public static bool IsGoodToGo;

        public static Exception LastException = null;

        public static string Operation = "";
        public static String Folder = "";
        public static Guid FolderId;
        public static String Name = "";
        public static Guid RId;
        public static string RunBookGuidString;
        public static DateTime JobLastMod;

        private static SMAOrchestratorConfig _config;
        private static OrchestratorCredentials _credentials;
        private static OrchestratorContext _smaorch;

        //Create  two(2) List one for the parameters name & values 
        public static List<String> Paramname;
        public static List<String> Paramvalue;

        public static Guid[] ParamGuids;
        public static int ParamsFound;
        private static String[] NewParamValues;
        private static String[] ParamNames;

        public static int TotalRunBooks;
        private static DateTime StampDateTime;
        public const string JobStatusSucess = "success"; // Status = 0;  
        public const string JobStatusWaring = "Warning"; // Status = 7;  //RunBookStatusFailure = 7, 
        public const string JobStatusFailed = "Failed"; //  Status = 6;  //RunBookStatusFailed = 6,       
        public const string JobStatusInProgress = "InProgress"; //Status = 8;  //RunBookStatusInProgress = 8,   
        public const string JobStatusRunning = "Running"; //  Status = 8;  //RunBookStatusInProgress = 8, 
        public const string JobStatusPending = "Pending"; // Status = 8;  //RunBookStatusInProgress = 8,    
        public const string JobStatusCompleted = "Completed"; //Status = 0;  //RunBookStatusCompleted = 0,   
        public const string JobStatusCanceled = "Canceled"; //Status = 9;  //RunBookStatusCancelled = 9,  


        public enum ExitCode
        {
            Success = 0,
            Error = 1,
            InvalidParameters = 3,
            RunBookStartError = 4,
            InvalidSettings = 10
        }

        public static int Main(string[] args)
        {
            return DoWork(args);

        }

        private static int DoWork(string[] args)
        {
            try
            {
                //Sets the method name for debugging purposes
                MethodName = "[DoWork]";
                DateTime dt = DateTime.Now;
                Console.WriteLine("{0} Starting MS Orchestrator connector at {1}", MethodName, dt);
                Status = 0;
                //Setting the program from the INI config file
                _config = new SMAOrchestratorConfig();
                if (!_config.LoadSettings())
                {
                    Status = (int)ExitCode.InvalidSettings;
                    return 1;
                }

                //only if the settigs are ok continue reading arguments
                if (Status == 0)
                {

                    if (args.Length == 0)
                    {
                        Status = 3; // returns error in parameters
                        Console.WriteLine("{0}: Critical Error: This application requires at least one parameter.", MethodName);
                        Console.WriteLine("{0}: Valid operations are: START,STATUS,STOP", MethodName);
                        Status = (int)ExitCode.InvalidParameters;
                    }
                    else //One or more araguments found
                    {
                        //ReadArguments checks if the comand help,username,password and domain are in the arguments 
                        _credentials = new OrchestratorCredentials();
                        if (!_credentials.ParseCredentials(args, _config)) return 1;

                        if (Status == 0)
                            Run(args);
                    }
                }
                else
                {
                    Console.WriteLine(MethodName + " Critical Error: Invalid settings. Please verify your settings.");
                    Status = ((int)ExitCode.InvalidSettings);
                }
            }
            catch (Exception e)
            {
                LastException = e;
                Console.WriteLine(" Error" + e);
                if (Status > 1 || Status == 0)
                    Status = ((int)ExitCode.Error);
            }
            finally
            {
                //change the status of the program to throw only Zero or One 
                if (Status > 1)
                    Status = ((int)ExitCode.Error);

            }


            //  Console.ReadKey();

            //Return the Exit Code
            Console.WriteLine("Status {0}", Status);
            return Status;

        }

        private static void Run(string[] args)
        {
            MethodName = "[Run]";
            //Creates a variable for validation purposes
            IsGoodToGo = false;

            //The debug starts after the context is created since it requires of a list of RunBooks GUID
            if ((SetSmaorch()))
            {
                GetRunBookGuid(args);
                if (Status.Equals(0) && (IsGoodToGo))
                {
                    IsGoodToGo = IsRunBookGuidValid(RId);
                    if (IsGoodToGo)
                    {
                        ////Only if Debug is on
                        //if (_config.Debug)
                        //{
                        //    //display all the GUID for the runbook if Debug mode is ON
                        //    IsDebugModeOn(_smaorch);
                        //    //Display of the parameters enter in the ReadArguments if debug
                        //    DisplayCurrentSettings();
                        //}
                        SetOperation(args);
                    }
                    else
                    {
                        Console.WriteLine(MethodName +
                                          "Critical Error: Invalid parameters!. Please verify your parameters.");
                        Status = ((int)ExitCode.InvalidParameters);
                    }
                }
                else
                {
                    Console.WriteLine(MethodName +
                                          " Critical Error: Could not find Runbook in Orchestrator.");
                    Status = ((int)ExitCode.InvalidParameters);
                }


            }
        }

        public static void SetOperation(string[] args)
        {
            MethodName = "[SetOperation]";
            //We are finally ready to verify the operation Either START or GET STATUS
            var operationParamValues = CommandLine(args, "op");
            foreach (String s in operationParamValues)
            {
                Operation = s;
            }

            //if the operation is to get-status 
            if ((Operation.Equals("status", StringComparison.OrdinalIgnoreCase)))
            {
                TrackRunBook();
            } // or if the start operation was define
            else if ((Operation.Equals("start", StringComparison.OrdinalIgnoreCase)))
            {
                StartJob(_smaorch, args);

            }
            else if ((Operation.Equals("stop", StringComparison.OrdinalIgnoreCase)))
            {
                StopJob(_smaorch);
            }
            else //the operation is not START,STOP or GET-STATUS
            {

                Console.WriteLine(MethodName +
                                  " Critical Error: The operation is not valid!");
                Status = ((int)ExitCode.InvalidParameters);
            }
        }



        //Gets the RunBookGuid from the folder or the Guid entered by user. Here we assign the RId wich is the Runbook GUID 
        public static void GetRunBookGuid(string[] args)
        {
            MethodName = "[GetRunBookGuid]";
            RunBookGuidString = GetRunBookGuidStringWithGuid(args);

            if ((string.IsNullOrEmpty(RunBookGuidString)))
                RId = GetRunBookGuidWithFolderName(args);
            else
                RId = new Guid(RunBookGuidString);

            if (RId.Equals(Guid.Empty))
            {
                Status = (int)ExitCode.InvalidSettings;
                Console.WriteLine("{0} Error: Please check Runbook GUID or Path.", MethodName);
                IsGoodToGo = false;
            }

        }

        public static string GetRunBookGuidStringWithGuid(string[] args)
        {

            MethodName = "[GetRunBookGuidStringWithGuid]";
            //Then we check if the parameter id was entered
            var runBookGuid = IsIdFound(args, "id");
            return runBookGuid;
        }


        public static bool SetSmaorch()
        {
            MethodName = "[SetSmaorch]";
            bool pass = false;

            //sets an orchestrator context
            _smaorch = SetOrchestratorContext();

            //check if the context has any RunBooks. If there is more than one Runbook GoodtoGo is set to True
            Status = IsContextValid(_smaorch);

            if (Status == 0)
                pass = true;

            return pass;
        }


        public static Guid GetRunBookGuidWithFolderName(string[] args)
        {
            MethodName = "[GetRunBookGuidWithFolderName]";
            //Gets the folder in case that -rp parameter was defined
            Folder = IsRunbookPathFound(args, "rp");
            Name = GetRunBookName(Folder);

            if ((!(String.IsNullOrEmpty(Folder))) && (!(String.IsNullOrEmpty(Name))))
                RId = GetRunBookGuidWithName(Name, Folder);
            else
                Console.WriteLine("{0} Could not find Path or Name. Please check your settings", MethodName);

            return RId;
        }



        public static Guid GetRunBookGuidWithName(string runBookName, string folder)
        {
            MethodName = "[GetRunBookGuidWithName]";
            var runBookGuid = new Guid();
            runBookGuid = Guid.Empty;
            if (_config.Debug)
                Console.WriteLine("{0} Looking for a Runbook GUID with Name: {1} ", MethodName, runBookName);

            //fixes the folder when there is not slashes at the begging
            if (!(folder.StartsWith("\\")))
                folder = "\\" + folder;

            if (_config.Debug)
                Console.WriteLine("{0} Runbook Folder Path: {1} ", MethodName, folder);
            var ocontext = SetOrchestratorContext();
            MethodName = "[GetRunBookGuidWithName]";

            int retries = 1;
            bool hasValues = false;
            // Setup Data Services query
            DataServiceQueryContinuation<Runbook> nextRunbookLink = null;

            try
            {

                var runbookQuery = (from runbook in ocontext.Runbooks
                                    where (runbook.Path.CompareTo(folder.Trim()) == 0)
                                    select runbook) as DataServiceQuery<Runbook>;


                // Run the initial query
                if (runbookQuery != null)
                {
                    var queryResponse = runbookQuery.Execute() as QueryOperationResponse<Runbook>;
                    do
                    {
                        if (queryResponse != null)
                        {
                            foreach (var runBook in queryResponse)
                            {
                                hasValues = true;
                                if (_config.Debug)
                                {
                                    Console.WriteLine("{0} Comparing folder: {1}", MethodName, runBook.Path);
                                }

                                if (runBook.Path.Equals(folder, StringComparison.OrdinalIgnoreCase))
                                {
                                    runBookGuid = runBook.Id;
                                }
                            }

                            if (!hasValues)
                            {
                                if (_config.Debug)
                                    Console.WriteLine("{0} Error connecting to Orchestrator WebService Retrying ({1}/10)...", MethodName, retries);
                                Thread.Sleep(2000);
                                retries++;
                                if (retries == 11)
                                {
                                    Console.WriteLine("{0} Exhausted retries connecting to Orchestrator WebService. Service unavailable or invalid Name/Path", MethodName);
                                    queryResponse = null;
                                }
                                else queryResponse = runbookQuery.Execute() as QueryOperationResponse<Runbook>;
                            }
                        }
                    } while ((!hasValues) && (queryResponse != null));

                    if ((_config.Debug) && (!(runBookGuid.Equals(Guid.Empty))))
                        Console.WriteLine("{0} Found Guid: {1}", MethodName, runBookGuid);
                }
                else
                {
                    Console.WriteLine("{0} Query was not built by Orchestrator", MethodName);
                }

            }
            catch (DataServiceQueryException ex)
            {
                Console.WriteLine("{0} Could not retrieve RunBook GUID. Please check your parameters...{1}", MethodName, ex);
                Status = (int)ExitCode.InvalidParameters;
            }


            return runBookGuid;

        }





        //Gets the last time stamp modify for the lastes job run for a RunbookGUID
        private static void GetLastStopJobStatus(Guid runBookId, out Guid runBookJobId, out DateTime jobLastMod, out String jobstatustxt)
        {
            MethodName = "[GetLastStopJobStatus]";
            var ocontext = SetOrchestratorContext();
            MethodName = "[GetLastStopJobStatus]";
            jobstatustxt = "";
            runBookJobId = new Guid();
            jobLastMod = new DateTime();

            var jobQuery = (from job in ocontext.Jobs
                            where ((job.RunbookId.CompareTo(runBookId) == 0) && ((job.Status == "Pending") || (job.Status == "Running")))
                            // where ((job.RunbookId == runBookId) && ((job.Status == "Pending") || (job.Status == "Running")))
                            orderby job.LastModifiedTime descending
                            select job) as DataServiceQuery<Job>;

            if (jobQuery.Count() > 0)
            {
                // Run the initial query
                QueryOperationResponse<Job> queryResponse = jobQuery.Execute() as QueryOperationResponse<Job>;
                var j = queryResponse.FirstOrDefault();
                jobstatustxt = j.Status;
                runBookJobId = j.Id;
                jobLastMod = j.LastModifiedTime;
            }
            else
            {
                Console.WriteLine(MethodName + "Error: No Started Jobs found to Stop. ");
            }

            if (runBookJobId == Guid.Empty)
            {
                Console.WriteLine(MethodName + "Error: Job Id was not found. ");
            }

            if (_config.Debug)
            {
                Console.WriteLine(MethodName + " Job status is " + jobstatustxt);
            }



        }



        private static void StopJob(OrchestratorContext smaorch)
        {

            MethodName = "[StopJob]";

            if (Status == 0)
            {

                Console.WriteLine(MethodName + " Stop command was define!");
                Console.WriteLine(MethodName + " The RunBook Id is " +
                                  RId.ToString());
                //Gets the Jod id
                Guid runBookJobId;
                DateTime jobLastMod;
                String jobstatustxt;
                GetLastStopJobStatus(RId, out runBookJobId, out jobLastMod, out jobstatustxt);

                MethodName = "[StopJob]";
                //Get the job
                var job = smaorch.Jobs.Where(j => j.Id.CompareTo(runBookJobId) == 0).SingleOrDefault();

                // Set the job status and save the change
                if (job != null)
                {

                    job.Status = "Canceled";
                    smaorch.UpdateObject(job);
                    smaorch.SaveChanges();
                    jobstatustxt = job.Status;
                    Console.WriteLine(MethodName + " After Stopping Job Status: " + jobstatustxt);


                }

                StampDateTime = DateTime.Now;
                Console.WriteLine(MethodName + " Exit Status {0} at Datetime stamp {1:G} ", Status, StampDateTime);
            }


        }



        private static void StartJob(OrchestratorContext smaorch, string[] args)
        {
            MethodName = "[StartJob]";
            // Setting the command to retreive the parameters from the app arguments.
            GetRunBookParameters(smaorch, args);
            MethodName = "[StartJob]";
            if (Status == 0)
            {
                Console.WriteLine(MethodName + " Start command was defined!");
                Console.WriteLine(MethodName + " The RunBook Id is " +
                                  RId.ToString());

                if ((Paramvalue.Count > 0) && (Paramname.Count > 0))
                    Status = StartAndTrack(smaorch, RId, Paramvalue, Paramname);
                else
                    Status = StartAndTrackRunBookWithoutParams(smaorch, RId);

            }

        }


        //We read all the parameters for Runbook here
        private static void GetRunBookParameters(OrchestratorContext smaorch, string[] args)
        {
            MethodName = "[GetRunBookParameters]";
            Paramname = new List<string>();
            Paramvalue = new List<string>();

            var parameterValues = CommandLine(args, "pa");
            //We assign the entire parameter value to a string and them we parse then
            //they will be stored in the paramnames and paramvalues global variables
            foreach (String s in parameterValues)
            {
                if (!String.IsNullOrEmpty(s))
                {
                    ParameterParse(s);
                }
            }
            MethodName = "[GetRunBookParameters]";

            //Get the total number of parameter for the RunBook
            var runBookParameters = GetRunBookParameters(smaorch, RId);

            if (runBookParameters.Equals(null))
            {
                Console.WriteLine(MethodName + " No parameters fetched by RunBook!");
            }
            else if (Paramname.Equals(null))
            {
                Console.WriteLine(MethodName + " No parameters entered by user!");
            }

        }

        private static void TrackRunBook()
        {
            Status = 0;
            MethodName = "[TrackJob]";

            Console.WriteLine(MethodName + " Status command was define!");
            Console.WriteLine(MethodName + " The RunBook Id is " + RId);
            //Here we get the status of a runbook 

            DateTime jobLastMod;
            Guid runBookJobId;
            string jobstatustxt;
            Status = GetJobStatus(RId, out jobLastMod, out runBookJobId, out jobstatustxt);

            //logs the status of the job
            Console.WriteLine(MethodName + " Status of the Job is " + Status);

            if (Status != 0)
            {
                Console.WriteLine("RunBook Job {0} was not successful", RId);
                Status = (int)ExitCode.Error;

            }

            Status = GetRunbookInstanceStatus(RId, jobLastMod, runBookJobId);

        }


        private static string IsIdFound(string[] args, String command)
        {
            MethodName = "[IsIdFound]";
            var idParamValues = CommandLine(args, command);
            var runBookGuid = "";

            foreach (var s in idParamValues)
            {
                runBookGuid = s;
            }
            return runBookGuid;
        }

        //GEts the folder in case that -rp parameter was defined
        //The parameter for Runbook Path is rp
        private static string IsRunbookPathFound(string[] args, String command)
        {
            MethodName = "[IsRunbookPathFound]";
            var folderParam = CommandLine(args, command);
            var runBookFolder = "";

            foreach (var s in folderParam)
            {
                runBookFolder = s;
            }
            return runBookFolder;
        }

        //Gets the Runbook name from the RunBook Path name
        private static string GetRunBookName(string runbookPath)
        {
            MethodName = "[GetRunBookName]";
            var runBookName = "";
            var split = runbookPath.Split(new Char[] { '\\' }, StringSplitOptions.RemoveEmptyEntries);
            var slength = split.Length;

            if (slength > 1)
            {
                runBookName = split[slength - 1];
                if (_config.Debug)
                    Console.WriteLine("{0}: Runbook Name is: {1}", MethodName, runBookName);
            }
            else
            {
                Console.WriteLine("{0}: Error getting Runbook Name. Please verify Path!", MethodName);
                Status = (int)ExitCode.InvalidParameters;
            }
            return runBookName;
        }


        private static List<String> CommandLine(IList<string> arguments, string parameter)
        {
            MethodName = "[CommandLine]";
            var spliter = new Regex(@"^-{1,2}|^/|=|:", RegexOptions.IgnoreCase | RegexOptions.Compiled);
            var remover = new Regex(@"^['""]?(.*?)['""]?$", RegexOptions.IgnoreCase | RegexOptions.Compiled);
            var argslength = arguments.Count;
            var values = new List<string>();
            var paramPosition = 0;
            var position = 0;
            var foundParam = false;

            foreach (var parts in arguments.Select(txt => spliter.Split(txt, 3)))
            {
                //If there is more than one part is because it found a potential PARAMETER
                if (parts.Length > 1)
                {
                    foreach (var s in parts.Where(s => parameter.Equals(s, StringComparison.OrdinalIgnoreCase)))
                    {
                        foundParam = true;
                        paramPosition = position;
                    }
                }
                //If a parameter was found then it gets the next arguments as a value
                if ((foundParam))
                {
                    //Determine the position of the value which is the next one after a parameter
                    if (paramPosition + 1 < argslength)
                    {
                        var valuePosition = paramPosition + 1;
                        //Remove the additional enclosed values from the parameter
                        var value = remover.Replace(arguments[valuePosition], "$1");
                        //Gets the value for the parameter 
                        if ((value != "") && (parameter != value) && (parameter != ""))
                        {
                            values.Add(value);
                        }
                        else
                        {
                            if ((string.IsNullOrEmpty(value)) || (string.IsNullOrEmpty(parameter)) || (parameter.Equals(value)))
                            {
                                //Log this error
                                Console.WriteLine(MethodName + " Invalid Parameters.");
                                Status = (int)ExitCode.InvalidParameters;
                            }
                        }
                        //Sets the FoundParam flag to false
                        foundParam = false;
                    }
                    else // In case the parameter value doesn't exits
                    {
                        Console.WriteLine(MethodName + " Invalid Parameters.");
                        Status = (int)ExitCode.InvalidParameters;
                    }
                }
                position++;
            }

            return values.ToList();
        }

        private static void ParameterParse(String parameters)
        {
            MethodName = "[ParameterParse]";


            if (string.IsNullOrEmpty(parameters))
            {
                throw new ArgumentNullException("parameters");
            }

            MethodName = "[ParameterParse]";
            var spliter = new Regex(@"^-{1,2}|^/|=|;", RegexOptions.IgnoreCase | RegexOptions.Compiled);
            var param = spliter.Split(parameters);
            //Paramname = new List<string>();
            //Paramvalue = new List<string>();

            //we loop the array and assume that odd positions are the name of the parameters and the
            //even positions are the values of the paramters
            for (var i = 0; i < param.Length; i++)
            {
                if ((param[i] != "") && (i % 2 == 0))
                {
                    Paramname.Add(param[i]);
                }
                if ((param[i] != null) && (i % 2 != 0))
                {
                    Paramvalue.Add(param[i]);
                }
            }


        }





        //This method verify Orchestrator context
        private static int IsContextValid(OrchestratorContext smaorch)
        {
            MethodName = "[IsContextValid]";

            try
            {
                // Create a Runbook variable in the new Orchestrator Context
                var runbooks = smaorch.Runbooks.FirstOrDefault();
                if (runbooks == null)
                    Status = 1;
                else
                    IsGoodToGo = true;
            }
            catch
            {

                Console.WriteLine(MethodName + " Critical Error: Context is invalid. Check your settings.");
                Status = 1;
            }
            return Status;
        }


        //This method verify the Runbook GUID exist in the Orchestrator context
        private static bool IsRunBookGuidValid(Guid runBookGuid)
        {
            MethodName = "[IsRunBookGUIDValid]";
            var isRunBookGuidValid = false;
            var ocontext = SetOrchestratorContext();
            MethodName = "[IsRunBookGUIDValid]";
            Console.WriteLine("{0} Looking for a RunBookGUID : {1} ", MethodName, runBookGuid);
            // Setup Data Services query
            DataServiceQueryContinuation<Runbook> nextRunbookLink = null;
            int retries = 1;
            bool hasValues = false;

            try
            {
                // Setup the query to retrieve the jobs
                var runbookQuery = (from runbook in ocontext.Runbooks
                                    where (runbook.Id.CompareTo(runBookGuid) == 0)
                                    select runbook) as DataServiceQuery<Runbook>;

                // Run the initial query
                if (runbookQuery != null)
                {
                    var queryResponse = runbookQuery.Execute() as QueryOperationResponse<Runbook>;
                    do
                    {


                        // Keep querying the next page of jobs until we reach the end
                        if (nextRunbookLink != null)
                        {
                            queryResponse = ocontext.Execute(nextRunbookLink) as QueryOperationResponse<Runbook>;
                        }
                        if (queryResponse != null)
                        {
                            foreach (var runBook in queryResponse)
                            {
                                hasValues = true;
                                if ((runBook.Id.CompareTo(runBookGuid) == 0))
                                {
                                    Console.WriteLine("{0} Found a RunBook GUID {1}", MethodName, runBook.Id);
                                    // Console.WriteLine("{0} RunBook folder {1}", MethodName, runBook.Folder);

                                    if (runBook.CheckedOutTime.Equals(null))
                                    {
                                        isRunBookGuidValid = true;
                                        Console.WriteLine(MethodName + " RunBook GUID is valid.");
                                        Status = (int)ExitCode.Success;
                                    }
                                    else
                                    {
                                        isRunBookGuidValid = false;
                                        Console.WriteLine(MethodName +
                                                          " Cannot process the request as Runbook is checked out. Checked out on:  " +
                                                          runBook.CheckedOutTime);
                                        Status = (int)ExitCode.Error;
                                    }

                                }

                            }
                        }

                        if (!hasValues)
                        {
                            if (_config.Debug)
                                Console.WriteLine("{0} Error connecting to Orchestrator WebService Retrying ({1}/10)...", MethodName, retries);
                            Thread.Sleep(2000);
                            retries++;
                            if (retries == 11)
                            {
                                Console.WriteLine("{0} Exhausted retries connecting to Orchestrator WebService. Service unavailable or invalid GUID", MethodName);
                                queryResponse = null;
                                hasValues = true;
                            }
                            else queryResponse = runbookQuery.Execute() as QueryOperationResponse<Runbook>;
                        }
                        else
                            nextRunbookLink = queryResponse.GetContinuation();
                    } while ((queryResponse != null) && (nextRunbookLink != null) && (isRunBookGuidValid == false) || !hasValues);
                }
                else
                {
                    Console.WriteLine("{0} Error: An error occurred building the query to search for Runbook!", MethodName);
                    Console.WriteLine("{0} Error: in the query {1}", MethodName, runbookQuery);
                    Status = (int)ExitCode.Error;
                }

                return isRunBookGuidValid;

            }
            catch (DataServiceQueryException ex)
            {
                Console.WriteLine("{0} Critical Error: {1} ", MethodName, ex);
                Status = (int)ExitCode.Error;
                return false;
            }

        }




        //This method returns the parameter of a RunBook within an Orchestrator context and RunBook Id
        private static List<String> GetRunBookParameters(OrchestratorContext ocontext, Guid runBookId)
        {
            // Details of runbook that we are going to run.
            var runBookParam = new List<string>();
            //Get the runbook parameters
            var parameters = ocontext.RunbookParameters.Where(runbookParam => (runbookParam.RunbookId.CompareTo(runBookId) == 0) && (runbookParam.Direction.Equals("In")));
            //  var parameters = ocontext.RunbookParameters.Where(runbookParam => runbookParam.RunbookId == runBookId && runbookParam.Direction == "In");

            foreach (var p in parameters)
            {
                runBookParam.Add(p.Name);
            }
            return runBookParam;
        }

        //Set the orchestrator context with a ServiceRootURI and credentials
        private static OrchestratorContext SetOrchestratorContext()
        {
            MethodName = "[SetOrchestratorContext]";
            var serviceRootUri = new Uri(_config.ServiceRootURI);
            var smaorch = new OrchestratorContext(serviceRootUri)
            {
                IgnoreResourceNotFoundException = true,
                IgnoreMissingProperties = true
            };
            //Added to avoid 404 response from webservice

            if ((String.IsNullOrEmpty(_credentials.Password)) || (String.IsNullOrWhiteSpace(_credentials.Password)) || (_credentials.Password.Length <= 0))
            {

                Console.WriteLine(MethodName + " Invalid Password.");
                Status = (int)ExitCode.InvalidParameters;
            }
            else
            {
                try
                {
                    var decryptionAlgorithm = new LengthBasedAlgorithm();
                    var decryptedPwd = decryptionAlgorithm.Decrypt(_credentials.Password);
                    smaorch.Credentials = new NetworkCredential(_credentials.Username, decryptedPwd, _credentials.Domain);

                    if (String.Equals(decryptedPwd, _credentials.Password))
                    {
                        Status = (int)ExitCode.InvalidParameters;
                    }
                }
                catch
                {

                    Status = (int)ExitCode.InvalidParameters;
                    Console.WriteLine("{0} Error occurred when trying to connect to the Orchestrator Context!", MethodName);
                }
            }


            //Console.WriteLine("The context is: {0}",smaorch.Timeout);
            return smaorch;
        }



        private static void ProcessParams(OrchestratorContext ocontext, Guid runBookId, List<string> runBookParameters, List<string> runBookParamnames)
        {

            MethodName = "[ProcessParams]";
            //build a dictionary with parameters key value pair
            //ParameterList dict = new ParameterList();
            //int index = 0;
            //foreach (var name in runBookParamnames)
            //{
            //    dict.Add(name, runBookParameters[index]);
            //    index++;
            //}

            var runbookparamaters = ocontext.RunbookParameters.Where(runbookParam => runbookParam.RunbookId == runBookId && runbookParam.Direction == "In");
            NewParamValues = new string[runbookparamaters.Count()];
            ParamNames = new string[runbookparamaters.Count()];
            ParamGuids = new Guid[runbookparamaters.Count()];

            // Makes sure the parameters exist in the Orchestrator context also that user send parameters name and values ~ This was requested by QA Kkanaru 02/06/15
            if (runBookParamnames.Count > 0 && runBookParameters.Count > 0)
            {
                int x = 0;
                foreach (var pitem in runbookparamaters)
                {
                    ParamNames[x] = pitem.Name;
                    ParamGuids[x] = pitem.Id;
                    //Verify that parameter passed by user exits in the orchestrator context
                    foreach (var name in runBookParamnames)
                    {
                        if (pitem.Name.Equals(name, StringComparison.CurrentCultureIgnoreCase))
                        {
                            NewParamValues[x] = runBookParameters[runBookParamnames.IndexOf(name)];
                        }
                    }

                    x++;
                }

            }
            else
            {
                Console.WriteLine("{0} No parameters fetched in the context!", MethodName);
            }

        }




        private static void ProcessParams(OrchestratorContext ocontext, Guid runBookId)
        {

            MethodName = "[ProcessParams]";
            // ParamGuids = new Guid[n.Length];
            var runbookparamaters = ocontext.RunbookParameters.Where(runbookParam => (runbookParam.RunbookId.CompareTo(runBookId) == 0) && runbookParam.Direction.Equals("In"));

            NewParamValues = new string[runbookparamaters.Count()];
            ParamNames = new string[runbookparamaters.Count()];
            ParamGuids = new Guid[runbookparamaters.Count()];

            // Makes sure the parameters exist in the Orchestrator context also that user send parameters name and values ~ This was requested by QA Kkanaru 02/06/15
            int x = 0;
            foreach (var pitem in runbookparamaters)
            {
                ParamNames[x] = pitem.Name;
                ParamGuids[x] = pitem.Id;
                x++;
            }
        }


        //This method builds a RunBook GUI and retreive parameters. If there is parameter will build the XML string with parameters values 
        // and them creates a job adds the runbook and save to context so it can start. If there is no parameters then it just create job and save.
        private static int StartAndTrack(OrchestratorContext ocontext, Guid runBookId, List<string> runBookParameters, List<string> runBookParamnames)
        {
            Status = 0;
            MethodName = "[StartAndTrack]";
            DateTime nowDateTime = DateTime.Now;
            var strParam = "<Data>";
            Guid newJobId = Guid.Empty;
            try
            {
                ProcessParams(ocontext, runBookId, runBookParameters, runBookParamnames);
                MethodName = "[StartAndTrack]";
                if (ParamNames.Length.Equals(ParamGuids.Length))
                {
                    for (var x = 0; x < ParamNames.Length; x++)
                    {
                        if (Guid.Empty.CompareTo(ParamGuids[x]) != 0)
                        {
                            strParam += "<Parameter><Name>" + ParamNames[x] + "</Name>";
                            strParam += "<ID>{" + ParamGuids[x] + "}</ID>";
                            strParam += "<Value>" + NewParamValues[x] + "</Value></Parameter>";
                        }

                    }
                    strParam += "</Data>";
                }
                else
                {
                    Console.WriteLine("{0} Error! Mismatched parameters.", MethodName);
                }
                // Create new job and assign runbook Id and parameters.
                var newjob = new Job { RunbookId = runBookId, Parameters = strParam };
                // Add newly created job.
                ocontext.AddToJobs(newjob);
                bool saved = false;
                int retries = 1;
                while (!saved && retries <= 10)
                {
                    try
                    {
                        ocontext.SaveChanges();
                        saved = true;
                    }
                    catch (Exception x)
                    {
                        Console.WriteLine("Orchestrator web service error creating job, retrying ({0}/10)", retries);
                        if (_config.Debug)
                            Console.WriteLine(x.ToString());
                        retries++;
                        Thread.Sleep(2000);
                    }
                }
                if (saved == false)
                    throw new Exception("Exhausted retries when trying to create new job.");
                newJobId = newjob.Id;
                Console.WriteLine("{0} Creating new Job in Orchestrator with GUID : {1}", MethodName, newJobId);

            }
            catch (Exception ex)
            {
                Console.WriteLine("{0} Error starting runbook: {1}", MethodName, ex);
                Status = 1;
            }

            return TrackJobId(runBookId, newJobId);

        }



        private static int TrackJobId(Guid runBookGuid, Guid jobGuid)
        {
            MethodName = "[TrackJobId]";
            DateTime jobLastMod;
            Guid empty = Guid.Empty;

            if ((!(runBookGuid.Equals(Guid.Empty))) && (!(jobGuid.Equals(Guid.Empty))))
            {
                Status = GetJobIdStatus(runBookGuid, jobGuid, out jobLastMod);
                if (Status != 0)
                {
                    Console.WriteLine("Error while searching for Job {0}", jobGuid);
                    Status = (int)ExitCode.RunBookStartError;
                    return Status;
                }

                Status = GetRunbookInstanceStatus(runBookGuid, jobLastMod, jobGuid);

            }
            else
            {
                Console.WriteLine("{0} Error tracking runbook. Job Guid is empty!", MethodName);
                Status = 1;
            }

            return Status;

        }



        //This method builds a RunBook GUI without any parameters. 
        // and them creates a job adds the runbook and save to context so it can start.
        private static int StartAndTrackRunBookWithoutParams(OrchestratorContext ocontext, Guid runBookId)
        {
            MethodName = "[StartAndTrackRunBookWithoutParams]";
            DateTime nowDateTime = DateTime.Now;
            //if the runbook has parameters we create the xml string without information
            var strParam = "<Data>";
            Guid newJobId = Guid.Empty;
            try
            {
                ProcessParams(ocontext, runBookId);
                MethodName = "[StartAndTrackRunBookWithoutParams]";
                if (ParamNames.Length > 0)
                {
                    for (var x = 0; x < ParamNames.Length; x++)
                    {

                        //if (Guid.Empty != ParamGuids[x])
                        if (!(ParamGuids[x].Equals(Guid.Empty)))
                        {
                            strParam += "<Parameter><Name>" + ParamNames[x] + "</Name>";
                            strParam += "<ID>{" + ParamGuids[x] + "}</ID>";
                            strParam += "<Value></Value></Parameter>";
                        }

                    }

                }
                strParam += "</Data>";
                Console.WriteLine("{0} Starting and Tracking RunBook without parameters.", MethodName);
                Status = 0;

                // Create new job and assign runbook Id and parameters.
                var newjob = new Job { RunbookId = runBookId, Parameters = strParam };
                ocontext.AddToJobs(newjob);
                bool saved = false;
                int retries = 1;
                while (!saved && retries <= 10)
                {
                    try
                    {
                        ocontext.SaveChanges();
                        saved = true;
                    }
                    catch (Exception x)
                    {
                        Console.WriteLine("Orchestrator web service error creating job, retrying ({0}/10)", retries);
                        if (_config.Debug)
                            Console.WriteLine(x.ToString());
                        retries++;
                        Thread.Sleep(2000);
                    }
                }

                if (saved == false)
                    throw new Exception("Exhausted retries when trying to create new job.");

                newJobId = newjob.Id;
                Console.WriteLine("{0} Creating new Job in Orchestrator with GUID : {1}", MethodName, newJobId);
            }
            catch (Exception ex)
            {
                Console.WriteLine("{0} Error Starting and Tracking RunBook without parameters: {1}", MethodName, ex);
                Status = 1;
            }

            return TrackJobId(runBookId, newJobId);

        }

        //This method finds the status of the Job and last time modify, them it looks for the status of the instance and 
        //compare time stamps to determine completion of trigger job
        private static int GetJobIdStatus(Guid runBookId, Guid runBookJobId, out DateTime jobLastMod)
        {
            MethodName = "[GetJobIdStatus]";
            Status = 0;
            jobLastMod = new DateTime();
            Guid runBookJobIdTarget = runBookJobId;
            String jobstatustxt = string.Empty;

            while ((jobstatustxt != "Completed") && (jobstatustxt != "Failed") && (jobstatustxt != "Canceled"))
            {
                Thread.Sleep(5000);
                Status = GetJobIdLastStatus(runBookId, runBookJobIdTarget, out jobLastMod, out jobstatustxt);
                JobLastMod = jobLastMod.ToUniversalTime();
            }

            if ((Status.Equals(6)) || (Status.Equals(9)))
            {
                Console.WriteLine(MethodName + "Job Id {0} cancelled or failed.", runBookJobId);
                Status = (int)ExitCode.RunBookStartError;
                return Status;
            }

            Console.WriteLine(MethodName + " Last Job Status {0} and last time stamp {1}", jobstatustxt, jobLastMod);

            return Status;
        }



        private static int GetJobIdLastStatus(Guid runBookId, Guid runBookJobId, out DateTime jobLastTimeUpdate, out String jobStatusTxt)
        {
            Status = 0;
            jobStatusTxt = "";
            jobLastTimeUpdate = new DateTime();
            var ocontext = SetOrchestratorContext();
            MethodName = "[GetJobIdLastStatus]";
            Console.WriteLine(MethodName + "Searching for Job id {0}", runBookJobId);

            DataServiceQueryContinuation<Job> nextJobLink = null;

            try
            {
                var jobQuery = (from job in ocontext.Jobs
                                //where job.RunbookId == runBookId
                                where (job.RunbookId.CompareTo(runBookId) == 0)
                                select job) as DataServiceQuery<Job>;

                if (jobQuery != null)
                {

                    var queryResponse = jobQuery.Execute() as QueryOperationResponse<Job>;

                    do
                    {
                        // Keep querying the next page of jobs until we reach the end
                        if (nextJobLink != null)
                        {
                            queryResponse = ocontext.Execute(nextJobLink) as QueryOperationResponse<Job>;
                            //  Console.WriteLine("{0}: New page ~ Quering the next 50 jobs....  ", MethodName);
                        }

                        if (queryResponse != null)
                        {
                            foreach (var j in queryResponse.Where(j => (j.Id.CompareTo(runBookJobId) == 0)))
                            {
                                jobStatusTxt = j.Status;
                                jobLastTimeUpdate = j.LastModifiedTime;

                            }
                        }

                    } while (queryResponse != null &&
                             ((nextJobLink = queryResponse.GetContinuation()) != null && (String.IsNullOrEmpty(jobStatusTxt))));

                    Status = GetJobStatusInt(jobStatusTxt);
                    if ((_config.Debug) && (!(jobLastTimeUpdate.Equals(JobLastMod))) && (!(String.IsNullOrEmpty(jobStatusTxt))))
                    {

                        Console.WriteLine("{0} Job status is {1}", MethodName, jobStatusTxt);
                        Status = GetJobStatusInt(jobStatusTxt);
                    }

                }

            }
            catch (DataServiceQueryException ex)
            {
                Console.WriteLine("{0} Job id {1} not found! Error {2}", MethodName, runBookJobId, ex);
            }

            return Status;

        }


        //This method finds the runbook GUID with the path of a Runbook.
        private static int GetRunbookInstanceStatus(Guid runBookId, DateTime jobLastMod, Guid jobId)
        {
            MethodName = "[GetRunbookInstanceStatus]";
            Status = 0;
            var rIdatetmp = new DateTime();
            var ocontext = SetOrchestratorContext();
            string statustxt = String.Empty;
            MethodName = "[GetRunbookInstanceStatus]";
            bool endloop = false;

            DataServiceQueryContinuation<RunbookInstance> nextRunbookInstanceLink = null;
            // Setup the query to retrieve the jobs
            Console.WriteLine(MethodName + "Searching instances for RunbookId {0} with last time stamp of {1}", runBookId, jobLastMod);


            try
            {
                var runbookInstancesQuery = (from runbookInstance in ocontext.RunbookInstances
                                             where (runbookInstance.RunbookId.CompareTo(runBookId) == 0)
                                             select runbookInstance) as DataServiceQuery<RunbookInstance>;

                var queryInstResponse = runbookInstancesQuery.Execute() as QueryOperationResponse<RunbookInstance>;

                do
                {

                    if (nextRunbookInstanceLink != null)
                    {
                        queryInstResponse = ocontext.Execute(nextRunbookInstanceLink) as QueryOperationResponse<RunbookInstance>;
                    }

                    if (queryInstResponse == null)
                    {
                        continue;
                    }

                    foreach (var runbookInstance in queryInstResponse.Where(runbookInstance => (runbookInstance.JobId == jobId)))
                    {
                        if (runbookInstance.CompletionTime != null)
                            rIdatetmp = runbookInstance.CompletionTime.Value;

                        rIdatetmp = rIdatetmp.ToUniversalTime();
                        jobLastMod = jobLastMod.ToUniversalTime();

                        var compdates = (rIdatetmp.Date.Equals(jobLastMod.Date)) && ((rIdatetmp.Hour.Equals(jobLastMod.Hour))) && ((rIdatetmp.Minute.Equals(jobLastMod.Minute)));
                        if (!compdates)
                        {
                            continue;
                        }
                        endloop = true;
                        if (_config.Debug)
                        {
                            Console.WriteLine(MethodName + " Runbook Instance creation Time was: " + runbookInstance.CreationTime);
                        }
                        //Gets the status of the Runbook
                        statustxt = runbookInstance.Status;
                        Status = GetJobStatusInt(statustxt);
                        if (_config.Debug)
                        {
                            Console.WriteLine(MethodName + " RunbookInstance: " + statustxt);
                        }

                        Console.WriteLine(MethodName + " RunbookInstance completion Time was: " + runbookInstance.CompletionTime);
                    }
                } while ((queryInstResponse != null) &&
                      ((nextRunbookInstanceLink = queryInstResponse.GetContinuation()) != null) &&
                      (endloop == false));


            }
            catch (DataServiceQueryException ex)
            {
                Console.WriteLine("{0} Error while query runbook instances for runbook id {1}. Error {2}", MethodName, runBookId, ex);

            }


            return Status;
        }

        //Gets the last time stamp modify for the lastes job run for a RunbookGUID
        private static int GetLastJobStatus(Guid runBookId, out Guid runBookJobId, out DateTime jobLastMod, out String jobstatustxt)
        {
            Status = 0;
            MethodName = "[GetLastJobStatus]";
            var ocontext = SetOrchestratorContext();
            MethodName = "[GetLastJobStatus]";

            jobstatustxt = "";
            runBookJobId = new Guid();
            jobLastMod = new DateTime();


            try
            {
                var jobQuery = (from job in ocontext.Jobs
                                where ((job.RunbookId.CompareTo(runBookId) == 0) && (job.Status != "Pending") && (job.Status != "Running"))
                                // where ((job.RunbookId == runBookId) && (job.Status != "Pending") && (job.Status != "Running"))
                                orderby job.LastModifiedTime descending
                                select job) as DataServiceQuery<Job>;

                if (jobQuery != null)
                {

                    // Run the initial query
                    QueryOperationResponse<Job> queryResponse = jobQuery.Execute() as QueryOperationResponse<Job>;
                    var j = queryResponse.FirstOrDefault();
                    jobstatustxt = j.Status;
                    runBookJobId = j.Id;
                    jobLastMod = j.LastModifiedTime;

                    if (jobstatustxt != "")
                    {
                        Status = GetJobStatusInt(jobstatustxt);
                        Console.WriteLine(MethodName + " Job status is " + jobstatustxt);
                    }

                }

            }
            catch (DataServiceQueryException ex)
            {
                Console.WriteLine("{0} Error while querying the Job id {1}. Error {2}", MethodName, runBookJobId, ex);
            }
            catch (NullReferenceException ex)
            {
                Console.WriteLine(" {0} Error while querying the Job id {1}. Error {2}", MethodName, runBookJobId, ex);
            }

            if (runBookJobId.CompareTo(Guid.Empty) == 0)
            {
                Console.WriteLine("{0} Error: Job Id was not found. ", MethodName);
            }


            return Status;
        }



        //This method returns the Last Job Status of a RunBook Instance
        private static int GetJobStatus(Guid runBookId, out DateTime jobLastMod, out Guid jobId, out string jobstatustxt)
        {
            MethodName = "[GetJobStatus]";
            Status = 0;
            jobstatustxt = string.Empty;
            Guid runbookJobId = new Guid();
            DateTime jobLastModDt = new DateTime();

            while (jobstatustxt != "Completed" && jobstatustxt != "Failed" && jobstatustxt != "Canceled")
            {
                Thread.Sleep(5000);
                Status = GetLastJobStatus(runBookId, out runbookJobId, out jobLastModDt, out jobstatustxt);
            }

            JobLastMod = jobLastModDt.ToUniversalTime();
            jobId = runbookJobId;
            jobLastMod = jobLastModDt;

            Console.WriteLine("{0} Exit Status {1} at Datetime stamp {2:G} ", MethodName, Status, StampDateTime);
            return Status;
        }


        //Change the status for WARNINGs on 03-28-2016 glastra
        private static int GetJobStatusInt(String jobstatus)
        {
            MethodName = "[GetJobStatusInt]";
            switch (jobstatus)
            {
                case JobStatusSucess:
                    Status = 0; break;
                case JobStatusWaring:
                    Status = 7; break;
                case JobStatusFailed:
                    Status = 6; break;
                case JobStatusInProgress:
                    Status = 8; break;
                case JobStatusRunning:
                    Status = 8; break;
                case JobStatusPending:
                    Status = 8; break;
                case JobStatusCompleted:
                    Status = 0; break;
                case JobStatusCanceled:
                    Status = 9; break;

                default:
                    return Status = 1;
            }

            // Console.WriteLine("{0} Exit Status {1} ", MethodName, Status);
            return Status;
        }

    }
}


