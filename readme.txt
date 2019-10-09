
using Microsoft.Crm.Sdk.Messages;
using Microsoft.Xrm.Sdk;
using Microsoft.Xrm.Sdk.Client;
using Microsoft.Xrm.Sdk.Query;
using Microsoft.Xrm.Tooling.Connector;
using Newtonsoft.Json;
using NLog;
using System;
using System.Collections.Generic;
using System.Configuration;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Net.Http.Headers;
using System.ServiceModel;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using System.Xml;

namespace CrmMigrator
{

    /// <summary>
    /// Интерфейс для retrieve функций получения данных о ресурсах
    /// </summary>
    public interface IServiceEquipment
    {
        EntityCollection getCollectionResouces(FetchExpression fetch);
		List<EntitySpecofiacation> getAllresources(EntityCollection collect);
		bool deleteAllSubgroups(List<EntitySpecofiacation> lstSpec);
		bool AssociateResources(List<EntitySpecofiacation> lstSpec);
		bool deleteSubgroupsByServiceId(List<EntitySpecofiacation> lstSpec,idService id);
		bool AssociateResourcesByServiceId(List<EntitySpecofiacation> lstSpec,idService id);
					
		
    }
	
	public interface IListResService
    {
        
		List<Entity> GetResources(Guid IdGroup);
		List<Entity> GetSubGroups(Guid IdGroup);
		bool clearAllResources();
				
    }
	

	public class EntitySpecifiacation // for retriev entities
	{
		Guid idService ;
		List<Entity> Resources;
		List<Entity> Subgroups;
		Giud idResSpec;
		Guid idSiteSettings;
		
		EntitySpecifiacation(Guid id)
		{
			
			idService =id; 	
		}
			
		
	}
	
	
	
	public class AppEquipment:IServiceEquipment
	{
		public IOrganizationService service;   
		
		public IListResService ListResService;
		
		public AppEquipment(IOrganizationService serv,IListResService addService)
        {
            service = serv;
        }
		
		
		EntityCollection GetCollectionResouces(FetchExpression fetch)
		{
            EntityCollection services = service.RetrieveMultiple(fetch);
					
		}
		
		List<EntitySpecofiacation> GetSpecificationService(EntityCollection collect)
		{
			list<EntitySpecofiacation> lst = new List<EntitySpecofiacation>();
			foreach (var curserv in collect)
            {
                addService.clearAllResources();
                counter++;
               // IEnumerable<Entity> lstresources = new List<Entity>();//list of all resources from service              
                //(listGroups as List<Entity>).Clear();
                Guid specGroupId = new Guid();
                Guid idService = new Guid();
                idService = curserv.Id;
                specGroupId = (Guid)curserv.GetAttributeValue<AliasedValue>("spec.groupobjectid").Value;
                
                logger.Debug($"### {counter}.Income in service name = {curserv.GetAttributeValue<string>("name")}");
                IEnumerable<Entity> lstresources = new List<Entity>(addService.GetResources(specGroupId));
                //lstresources = RetrieveRecursiveResourceByResourceGroup(specGroupId);//filled "grp" Dictionary
                IEnumerable<Entity> allGroups =new List<Entity>(addService.GetSubGroups()); 
                if (allGroups.Count() > 0)
                {
                    logger.Debug($"-OK- Service {curserv.GetAttributeValue<string>("name")} not null");
                    counterReadServices++;
                    counterReadGroup += allGroups.Count();
                    
					EntitySpecifiacation entry  = new EntitySpecifiacation(curserv.Id);
                    if (lstresources.Count() > 0)
                    {

                        entry.Resources = lstresources;
                        counterReadResources += lstresources.Count();
                    }

                    entry.Subgroups= allGroups;
                    entry.idResSpec = specGroupId;                   
                    Guid idSiteSetting = new Guid();
                    idSiteSetting =(Guid) curserv.GetAttributeValue<AliasedValue>("gm_sitesetting.gm_sitesettingid").Value;
                    entry.idSiteSetting = idSiteSetting;
                    lst.add(entry);
                    logger.Debug($"+++  service name = {curserv.GetAttributeValue<string>("name")}  - saved in memory.");

                }else logger.Debug($"-NO- Service {curserv.GetAttributeValue<string>("name")} is NULL");

            }

            if (dictEntry.Count() > 0)
            {
                state = true;
                logger.Debug($"***///      FINAL READ  ////***.");
                return true;
            }
            else
            {
                logger.Debug($"***///      All services is free  ////***.");
                return false;
            }			
			
		}
		
		//////////////////
		bool AssociateResourcesByServiceId(List<EntitySpecofiacation> lstSpec,idService id)
		{
			 string xmlnew = "";
            int CountResources = 0;
            logger.Debug($"***TRY write serviceid={idService}");
            if (state == false || idService ==null) return false;
            IEnumerable<Entity> lstres = dictEntry[idService].GetDictResources(idService);
           
            Entity resgroup = service.Retrieve("constraintbasedgroup", dictEntry[idService].resSpecGruop, new ColumnSet(new string[] { "constraints" }));
            if (!string.IsNullOrEmpty(xml))
            {
                xmlnew = xml;
            }else xmlnew = generateXmlConstraint(lstres);
            if (lstres != null)
            {
                CountResources = lstres.Count();
            }
            
            var odlConstr = resgroup.GetAttributeValue<string>("constraints");
            logger.Debug($"service id={idService} old value constraint //**//{odlConstr}//**//");
            logger.Debug($"#Count of resources={CountResources}. Value new constraint  = ###{xmlnew}###");
            try
            {
                

                resgroup["constraints"] = xmlnew;
                service.Update(resgroup);
                logger.Debug($"{counterWritedServices+1}. #++End write constraint service id={idService}");
                
                //clear constraint for SubGroupds
                IEnumerable<Entity> lstAllGroups = dictEntry[idService].GetDictGroups(idService);
                foreach(Entity grp in lstAllGroups)
                {
                    logger.Debug($"Begin clear subgrup. group id ={grp.Id} and name={grp.GetAttributeValue<string>("name")} ;  serviceid ={idService}");
                    Entity entGroup = service.Retrieve("constraintbasedgroup", grp.Id, new ColumnSet(new string[] { "constraints" }));
                    string oldConstr = entGroup.GetAttributeValue<string>("constraints");
                    logger.Debug($"Old constraints for group ={grp.Id} and name={grp.GetAttributeValue<string>("name")} ;  serviceid ={idService} ; constraints {oldConstr}");
                    entGroup["constraints"] = generateXmlConstraint(null);
                    service.Update(entGroup);
                    logger.Debug($"++End clear subgrup. group id ={grp.Id} and name ={grp.GetAttributeValue<string>("name")}  ; serviceid={idService}");
                }

                //Clear reference sitesetting to deleted group
                logger.Debug($"--Start clear sitesetting reference with id = {dictEntry[idService].idSiteSetting}  ; serviceid={idService}");
                Entity entSiteSetting = service.Retrieve("gm_sitesetting", dictEntry[idService].idSiteSetting, new ColumnSet(new string[] { "gm_constraintbasedgroupid" }));
                entSiteSetting["gm_constraintbasedgroupid"] = null;
                service.Update(entSiteSetting);
                logger.Debug($"++End clear sitesetting reference with id = {dictEntry[idService].idSiteSetting}  ; serviceid{idService}");
                logger.Debug($"***END try write serviceid={idService}");
                return true;
            }

            
             

            catch (Exception ex)
            {
                logger.Error($"ERROR WRITE SERVICE!! with service {idService} with messsage {ex.Message} and stack trace {ex.StackTrace}");
                return false;
                //throw new Exception($"Generated exception on the write service {idService} with messsage {ex.Message} and stack trace {ex.StackTrace}");
            }
			
		}
	
	}
	
	
	


    /// <summary>
    /// Класс обортка для рекурсивных функций поиска групп и ресурсов
    /// </summary>
    /// <param name="serv"></param>
    public class ServiceEquipmentAdditional : IListResService
    {
        public IOrganizationService service;
		IEnumerable<Entity> lstgrp = new List<Entity>();		
        List<Entity> GetResources(Guid IdGroup)
		{
			return RetrieveRecursiveResourceByResourceGroup(specGroupId,"",0);
		}
		
		
		List<Entity> GetSubGroups(Guid IdGroup)
		{
			return lstgrp;
			
		}
	   
	    
		     
        
		
		public int MaxDepthChildResourceGroup { get
            {
                return 5;
            }
        }

        public IEnumerable<Entity> listGroups { get
            {
                return lstgrp;
            }
            set
            {
                (lstgrp as List<Entity>).AddRange(value);
            }
        }



       
        public ServiceEquipmentAdditional(IOrganizationService serv)
        {
            service = serv;
        }

        /// <summary>
        /// Возвращает ресурсы входящие в группу ресурсов
        /// </summary>
        /// <param name="resourceGroupId">Идентификатор группы ресурсов</param>
        /// <param name="objectTypeCode">Получить ресусурсы определенного типа. 
        /// Если не указана, будут возвращены ресурсы всех типов</param>
        /// <returns>Type: IEnumerable<Md.Resource> Коллекция сущностей (ResourceGroup)</returns>
        public  IEnumerable<Entity> RetrieveByGroupResource(Guid resourceGroupId, string objectTypeCode = "")
        {
            // TraceIn($"ResourceGroupId = {resourceGroupId}");

            var request = new RetrieveByGroupResourceRequest()
            {
                ResourceGroupId = resourceGroupId,
                Query = new QueryExpression()
                {
                    NoLock = true,
                    EntityName = "resource",
                    ColumnSet = new ColumnSet(true)
                }
            };

            if (!string.IsNullOrEmpty(objectTypeCode))
            {
                (request.Query as QueryExpression).Criteria
                    .AddCondition("objecttypecode", ConditionOperator.Equal, objectTypeCode);
            }

            var response = (RetrieveByGroupResourceResponse)service.Execute(request);

            IEnumerable<Entity> resources = response.EntityCollection.Entities
                    //.Select(x => x.ToEntity<Entity>())
                    .ToArray();

            return resources;
        }

        /// <summary>
        /// Возвращает вложенные группы из группы ресурсов
        /// </summary>
        /// <param name="resourcegroupid">Идентификатор Родительской группы ресурсов</param>
        /// <returns>Type: IEnumerable<Md.ResourceGroup> Коллекция сущностей (ResourceGroup)</returns>
        public IEnumerable<Entity> RetrieveSubGroupsResourceGroup(Guid resourcegroupid)
        {
            // TraceIn($"ResourceGroupId = {resourcegroupid}");

            var request = new RetrieveSubGroupsResourceGroupRequest()
            {
                ResourceGroupId = resourcegroupid,
                Query = new QueryExpression() { EntityName = "resourcegroup", ColumnSet = new ColumnSet(new string[] { "name" }), NoLock = true }
            };

            var response = (RetrieveSubGroupsResourceGroupResponse)service.Execute(request);

            IEnumerable<Entity> groups = response.EntityCollection.Entities
                    // .Select(x => x.ToEntity<Entity>())
                    .ToArray();

            return groups;

        }



        public IEnumerable<Entity> RetrieveRecursiveResourceByResourceGroup(Guid resourceGroupId, string resourcetypecode = "", int depth = 0)
        {
            if (depth == MaxDepthChildResourceGroup)
            {
                //  Trace("Глубина поиска ресурсов в группе дотсигла максимального значения");
                //return Array.Empty<Md.Resource>();
            }
            // Получение списка Equipments
            List<Entity> resources = new List<Entity>();
            IEnumerable<Entity> firstLevelResources = RetrieveByGroupResource(resourceGroupId, resourcetypecode);
            resources.AddRange(firstLevelResources);
           
            IEnumerable<Entity> childGroups = RetrieveSubGroupsResourceGroup(resourceGroupId);
            if(childGroups != null)
            {
                listGroups = childGroups;
                foreach (var childGroup in childGroups)
                {
                    var childRes = RetrieveRecursiveResourceByResourceGroup(childGroup.Id, resourcetypecode, depth++);
                    resources.AddRange(childRes);
                }
            }
            

            return resources;
        }

        public bool clearAllResources()
        {
            if (lstgrp.Count() > 0)
            {
               (lstgrp as List<Entity>).Clear();
                
            }
            return true;

        }
    }



    /// <summary>
    /// Гравный класс программы
    /// </summary>
    public class Program
    {
        private  Dictionary<Guid,Entry> dictEntry = new Dictionary<Guid, Entry>(); //save guid grop service,list resources and constraint      
        private  bool state = false; //state of fill data     
       // private  int MaxDepthChildResourceGroup = 5;
        public  IOrganizationService service;
        public  List<Entity> listGroups = new List<Entity>();//Extracted when recursive get resources
        public static Logger logger = LogManager.GetCurrentClassLogger();
        public int counterDeletedGroup=0;
        public int counterReadGroup = 0;
        public int counterReadResources = 0;
        public int counterReadServices = 0;
        public int counterWritedServices = 0;     
        public Program(IOrganizationService serv)
        {
            service = serv;
        }
        public Dictionary<Guid, Entry> getAllEntries()
        {
            if (dictEntry == null) return null;
            else
                return dictEntry;
        }

        static void Main(string[] args)
        {
           

        }


       

        public void WriteStatisticsToLog()
        {
            if (state == false) {
                logger.Debug($"First you must call getAllEntries()");
            }
            else
            {
                logger.Debug($"Read services {counterReadServices}");
                logger.Debug($"Read subgroups {counterReadGroup}");
                logger.Debug($"Read resources {counterReadResources}");
                logger.Debug($"Write services {counterWritedServices}");
                logger.Debug($"Delete subgroups {counterDeletedGroup}");
            }
            
            



        }

        
        public bool  getAllResources(IRetrieveble wrapperRetrieveResFunc)
        {
            if (state == true) return true;



            /////////////////////////By ID////////////////////

            //var fetchData = new
            //{
            //    serviceid = "36611a3b-bfe1-e811-aab0-005056b42e08",
            //    constraints = "%false%"
            //};


            //var fetchXml = $@"
            //    <fetch top='5'>
            //      <entity name='service'>
            //        <attribute name='name'/>
            //         <filter>
            //             <condition attribute='serviceid' operator='eq' value='{fetchData.serviceid/*f5ffb322-d576-e711-80d0-005056b40c72*/}'/>
            //         </filter>

            //        <link-entity name='resourcespec' from='resourcespecid' to='resourcespecid' link-type='inner' alias='spec'>
            //            <attribute name='resourcespecid'/>
            //            <attribute name='groupobjectid'/>
            //            <link-entity name='constraintbasedgroup' from='constraintbasedgroupid' to='groupobjectid' alias='constraintbasedgroup'>
            //            <attribute name='constraints'/>
            //            <filter>
            //              <condition attribute='constraints' operator='not-like' value='{fetchData.constraints/*%false%*/}'/>
            //            </filter>
            //          </link-entity>
            //        </link-entity>
            //        <link-entity name='gm_sitesetting' from='gm_serviceid' to='serviceid' alias='gm_sitesetting'>
            //          <attribute name='gm_sitesettingid' />
            //        </link-entity>

            //      </entity>
            //    </fetch>";
            /////////////////////////Allll////////////////////////////////////
            var fetchData = new
            {
                constraints = "%false%"
            };

            var fetchXml = $@"
                <fetch top='3500'>
                  <entity name='service'>
                    <attribute name='name'/>

                    <link-entity name='resourcespec' from='resourcespecid' to='resourcespecid' link-type='inner' alias='spec'>
                        <attribute name='resourcespecid'/>
                        <attribute name='groupobjectid'/>
                        <link-entity name='constraintbasedgroup' from='constraintbasedgroupid' to='groupobjectid' alias='constraintbasedgroup'>
                        <attribute name='constraints'/>
                        <filter>
                          <condition attribute='constraints' operator='not-like' value='{fetchData.constraints/*%false%*/}'/>
                        </filter>
                      </link-entity>
                    </link-entity>
                    <link-entity name='gm_sitesetting' from='gm_serviceid' to='serviceid' alias='gm_sitesetting'>
                      <attribute name='gm_sitesettingid' />
                         <filter>
                            <condition attribute='gm_constraintbasedgroupid' operator='not-null' />
                         </filter>
                    </link-entity>

                  </entity>
                </fetch>";

            EntityCollection services = service.RetrieveMultiple(new FetchExpression(fetchXml));


            logger.Debug($"**********************************RUN CRM MIGRATOR DATA************************************");
           
            int counter = 0;
            foreach (var curserv in services.Entities)
            {
                wrapperRetrieveResFunc.clearAllResources();
                counter++;
               // IEnumerable<Entity> lstresources = new List<Entity>();//list of all resources from service              
                //(listGroups as List<Entity>).Clear();
                Guid specGroupId = new Guid();
                Guid idService = new Guid();
                idService = curserv.Id;
                specGroupId = (Guid)curserv.GetAttributeValue<AliasedValue>("spec.groupobjectid").Value;
                
                logger.Debug($"### {counter}.Income in service name = {curserv.GetAttributeValue<string>("name")}");
                IEnumerable<Entity> lstresources = new List<Entity> (wrapperRetrieveResFunc.RetrieveRecursiveResourceByResourceGroup(specGroupId,"",0));//filled "grp" Dictionary
                //lstresources = RetrieveRecursiveResourceByResourceGroup(specGroupId);//filled "grp" Dictionary
                IEnumerable<Entity> allGroups = new List<Entity>(wrapperRetrieveResFunc.listGroups); 
                if (allGroups.Count() > 0)
                {
                    logger.Debug($"-OK- Service {curserv.GetAttributeValue<string>("name")} not null");
                    counterReadServices++;
                    counterReadGroup += allGroups.Count();
                    Entry entry = new Entry();
                    if (lstresources.Count() > 0)
                    {

                        entry.setDictResources(idService, lstresources);
                        counterReadResources += lstresources.Count();
                    }

                    entry.setDictGroups(idService, allGroups);
                    entry.resSpecGruop = specGroupId;                   
                    Guid idSiteSetting = new Guid();
                    idSiteSetting =(Guid) curserv.GetAttributeValue<AliasedValue>("gm_sitesetting.gm_sitesettingid").Value;
                    entry.idSiteSetting = idSiteSetting;
                    dictEntry[idService] = entry;
                    logger.Debug($"+++  service name = {curserv.GetAttributeValue<string>("name")}  - saved in memory.");

                }else logger.Debug($"-NO- Service {curserv.GetAttributeValue<string>("name")} is NULL");

            }

            if (dictEntry.Count() > 0)
            {
                state = true;
                logger.Debug($"***///      FINAL READ  ////***.");
                return true;
            }
            else
            {
                logger.Debug($"***///      All services is free  ////***.");
                return false;
            }
        }



       

        /// <summary>
        /// Получает список групп по Id сервиса
        /// </summary>
        /// <param name="idService"></param>
        /// <returns></returns>
        public IEnumerable<Entity> getGroupsByService(Guid idService)
        {
            if (state == false || idService == null) return null;
            IEnumerable<Entity> itemGrp = dictEntry[idService].GetDictGroups(idService);
            foreach(var ent in itemGrp)
            {
                logger.Debug($" Get groups of service with id {idService}. Include group with name= {ent.GetAttributeValue<string>("name")} with id {ent.Id}");
            }
            return itemGrp;
        }


       


        /// <summary>
        /// Удаляет группу constraintbasedgroup по ID сервиса
        /// </summary>
        /// <param name="idService"></param>
        /// <returns></returns>
        public bool deleteGroupByServiceId(Guid idService)
        {
            if (idService == null) return false;
            IEnumerable<Entity> currentGroup = getGroupsByService(idService);
            foreach (Entity grpEnt in currentGroup)
            {
                try
                {                
                    logger.Debug($"--Start delete group id={grpEnt.Id} with name = {grpEnt.GetAttributeValue<string>("name")} ");                
                        
                    service.Delete("constraintbasedgroup", grpEnt.Id);
                    counterDeletedGroup++;
                    
                    logger.Debug($"++End delete group id={grpEnt.Id} with name = {grpEnt.GetAttributeValue<string>("name")}");
                }
                catch(Exception ex)
                {

                    logger.Error($"++ERROR .Group NO DELETED!!!! delete group id={grpEnt.Id} with name = {grpEnt.GetAttributeValue<string>("name")}");
                    //throw new Exception($"Could not delete group id {grpEnt.Id} service id {idService}.Stack{ex.StackTrace} ");
                }
                
            }
            return true;

        }


        /// <summary>
        /// Rewrite constraints for hiddem main group service 
        /// and clear constraints for all subgroups
        /// </summary>
        /// <param name="idService"></param>
        /// <param name="xml"></param>
        /// <returns></returns>
        public  bool writeServiceConstraint(Guid idService,string xml="")
        {
            string xmlnew = "";
            int CountResources = 0;
            logger.Debug($"***TRY write serviceid={idService}");
            if (state == false || idService ==null) return false;
            IEnumerable<Entity> lstres = dictEntry[idService].GetDictResources(idService);
           
            Entity resgroup = service.Retrieve("constraintbasedgroup", dictEntry[idService].resSpecGruop, new ColumnSet(new string[] { "constraints" }));
            if (!string.IsNullOrEmpty(xml))
            {
                xmlnew = xml;
            }else xmlnew = generateXmlConstraint(lstres);
            if (lstres != null)
            {
                CountResources = lstres.Count();
            }
            
            var odlConstr = resgroup.GetAttributeValue<string>("constraints");
            logger.Debug($"service id={idService} old value constraint //**//{odlConstr}//**//");
            logger.Debug($"#Count of resources={CountResources}. Value new constraint  = ###{xmlnew}###");
            try
            {
                

                resgroup["constraints"] = xmlnew;
                service.Update(resgroup);
                logger.Debug($"{counterWritedServices+1}. #++End write constraint service id={idService}");
                
                //clear constraint for SubGroupds
                IEnumerable<Entity> lstAllGroups = dictEntry[idService].GetDictGroups(idService);
                foreach(Entity grp in lstAllGroups)
                {
                    logger.Debug($"Begin clear subgrup. group id ={grp.Id} and name={grp.GetAttributeValue<string>("name")} ;  serviceid ={idService}");
                    Entity entGroup = service.Retrieve("constraintbasedgroup", grp.Id, new ColumnSet(new string[] { "constraints" }));
                    string oldConstr = entGroup.GetAttributeValue<string>("constraints");
                    logger.Debug($"Old constraints for group ={grp.Id} and name={grp.GetAttributeValue<string>("name")} ;  serviceid ={idService} ; constraints {oldConstr}");
                    entGroup["constraints"] = generateXmlConstraint(null);
                    service.Update(entGroup);
                    logger.Debug($"++End clear subgrup. group id ={grp.Id} and name ={grp.GetAttributeValue<string>("name")}  ; serviceid={idService}");
                }

                //Clear reference sitesetting to deleted group
                logger.Debug($"--Start clear sitesetting reference with id = {dictEntry[idService].idSiteSetting}  ; serviceid={idService}");
                Entity entSiteSetting = service.Retrieve("gm_sitesetting", dictEntry[idService].idSiteSetting, new ColumnSet(new string[] { "gm_constraintbasedgroupid" }));
                entSiteSetting["gm_constraintbasedgroupid"] = null;
                service.Update(entSiteSetting);
                logger.Debug($"++End clear sitesetting reference with id = {dictEntry[idService].idSiteSetting}  ; serviceid{idService}");
                logger.Debug($"***END try write serviceid={idService}");
                return true;
            }

            
             

            catch (Exception ex)
            {
                logger.Error($"ERROR WRITE SERVICE!! with service {idService} with messsage {ex.Message} and stack trace {ex.StackTrace}");
                return false;
                //throw new Exception($"Generated exception on the write service {idService} with messsage {ex.Message} and stack trace {ex.StackTrace}");
            }
            

        }



        public  string  GetAppointments(Guid idService,string serv= "https://msk-d365dev-web.nbn-holding.ru", string nameOrg= "nbnh-dev")
        {
            logger.Debug($" GetAppointments Start with serviceid={idService}");
            if (idService == null) return string.Empty;
            string defsrv = serv;
            var uri = new Uri(defsrv);
            string rez = "";
            double  timeoutSecons = 60;

            var credentialsCache = new CredentialCache { { uri, "NTLM", CredentialCache.DefaultNetworkCredentials } };
            var handler = new HttpClientHandler
            {
                Credentials = credentialsCache
                //,
                //PreAuthenticate = true
                // UseProxy = true,
                // Proxy = new WebProxy("http://localhost:8888", false)
            };
            logger.Debug($"GetAppointments create handler and cache with serviceid={idService}");

            using (var client = new HttpClient(handler, disposeHandler: true))
            {
                if (client == null) throw new Exception("Httpclient is null");
                client.BaseAddress = uri;
                //client.Timeout = new TimeSpan(0, 0, 10);
                client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
                string urlcrm = $"{nameOrg}/api/data/v8.2/Search(AppointmentRequest=@p1)?@p1={{\"Direction\":\"0\",\"Duration\":\"180\",\"NumberOfResults\":\"1000\",\"ServiceId\":\"{idService}\",\"SearchWindowStart\":\"2019-01-01T06:00:00%2b03:00\",\"SearchWindowEnd\":\"2019-10-01T06:00:00%2b03:00\",\"UserTimeZoneCode\":\"145\"}}";
                logger.Debug($"GetAppointments ser url serviceid={idService}");
                // try
                // {

                TimeSpan timeout = new TimeSpan();
                timeout = TimeSpan.FromSeconds(timeoutSecons);

         
                        DateTime startedTime = DateTime.Now;

                try
                {
                    logger.Debug($"GetAppointments start get response serviceid={idService}");
                    //response = await request.ExecuteAsync(cancelAfterDelay.Token);
                    Task<HttpResponseMessage> response = client.GetAsync(urlcrm /*, cancelAfterDelay.Token*/);
                    logger.Debug($"GetAppointments response is get serviceid={idService}");
                    if (response == null)
                    {
                        logger.Error($"GetAppointments response is NULL serviceid={idService}");
                        return "";
                    }
                    if (!response.Wait(timeout))
                    {
                        throw new TimeoutException($"Exception timeout with url  {urlcrm}");
                    }

                    rez = response.Result.Content.ReadAsStringAsync().Result;
                    DateTime endTime = DateTime.Now;
                    TimeSpan timeProcess = (endTime - startedTime);
                    //httpClient.Dispose();
                    logger.Debug($"GetAppointments read response content OK serviceid={idService} . Due=|{timeProcess.TotalSeconds}| seconds");
                }


                catch (TimeoutException ex)
                {
                    logger.Error($"GetAppointments TIMEOUT  in serviceid={idService} and URL = {serv+ urlcrm} ");
                    throw new TimeoutException("Timeout in get service idService {idService}");
                
                }
 
            }
            return rez;

          
        }
    


        /// <summary>
        /// Генерирует xml строку constraint для группы
        /// </summary>
        /// <param name="resources"></param>
        /// <returns></returns>
        private  string generateXmlConstraint(IEnumerable<Entity> resources)
        {
            string head = "<Constraints><Constraint><Expression><Body>";
            string foot = "</Body><Parameters><Parameter name=\"resource\" /></Parameters></Expression></Constraint></Constraints>";
            string body = "false";
            if (resources != null)
            {
                List<string> lst = new List<string>();
                foreach (Entity val in resources)
                {

                    lst.Add($"resource[\"Id\"] == {{{val.Id}}}");
                    body = String.Join("||", lst.ToArray());
                }
            }
                                
            var xml = head + body + foot;
            return xml;
        }

        

        /// <summary>
        /// Метод для записи в лог приложения
        /// </summary>
        /// <param name="str"></param>
        /// <param name="level"></param>
        public void WriteToLog(string str, int level = 0)
        {
            switch (level)
            {
                case 0:
                    logger.Debug($"{str}");
                    break;
                case 1:
                    logger.Warn($"{str}");
                    break;

            }

        }
        ///// <summary>
        ///// Возвращает ресурсы входящие в группу ресурсов
        ///// </summary>
        ///// <param name="resourceGroupId">Идентификатор группы ресурсов</param>
        ///// <param name="objectTypeCode">Получить ресусурсы определенного типа. 
        ///// Если не указана, будут возвращены ресурсы всех типов</param>
        ///// <returns>Type: IEnumerable<Md.Resource> Коллекция сущностей (ResourceGroup)</returns>
        //public  IEnumerable<Entity> RetrieveByGroupResource(Guid resourceGroupId, string objectTypeCode = "")
        //{
        //    // TraceIn($"ResourceGroupId = {resourceGroupId}");

        //    var request = new RetrieveByGroupResourceRequest()
        //    {
        //        ResourceGroupId = resourceGroupId,
        //        Query = new QueryExpression()
        //        {
        //            NoLock = true,
        //            EntityName = "resource",
        //            ColumnSet = new ColumnSet(true)
        //        }
        //    };

        //    if (!string.IsNullOrEmpty(objectTypeCode))
        //    {
        //        (request.Query as QueryExpression).Criteria
        //            .AddCondition("objecttypecode", ConditionOperator.Equal, objectTypeCode);
        //    }

        //    var response = (RetrieveByGroupResourceResponse)service.Execute(request);

        //    IEnumerable<Entity> resources = response.EntityCollection.Entities
        //            //.Select(x => x.ToEntity<Entity>())
        //            .ToArray();

        //    return resources;
        //}

        ///// <summary>
        ///// Возвращает вложенные группы из группы ресурсов
        ///// </summary>
        ///// <param name="resourcegroupid">Идентификатор Родительской группы ресурсов</param>
        ///// <returns>Type: IEnumerable<Md.ResourceGroup> Коллекция сущностей (ResourceGroup)</returns>
        //public  IEnumerable<Entity> RetrieveSubGroupsResourceGroup(Guid resourcegroupid)
        //{
        //    // TraceIn($"ResourceGroupId = {resourcegroupid}");

        //    var request = new RetrieveSubGroupsResourceGroupRequest()
        //    {
        //        ResourceGroupId = resourcegroupid,
        //        Query = new QueryExpression() { EntityName = "resourcegroup", ColumnSet = new ColumnSet(new string[]{ "name"}), NoLock = true }
        //    };

        //    var response = (RetrieveSubGroupsResourceGroupResponse)service.Execute(request);

        //    IEnumerable<Entity> groups = response.EntityCollection.Entities
        //            // .Select(x => x.ToEntity<Entity>())
        //            .ToArray();

        //    return groups;

        //}



        


        /// <summary>
        /// Вспомогательный класс для хранения данных ресурса, группы и спецификации по сервису
        /// </summary>
        public class Entry
        {
            private Dictionary<Guid, IEnumerable<Entity>> DictResources = new Dictionary<Guid, IEnumerable<Entity>>();
            private Dictionary<Guid, IEnumerable<Entity>> DictGroups = new Dictionary<Guid, IEnumerable<Entity>>();
            public Guid resSpecGruop { get; set;}
            public Guid idSiteSetting { get; set; }

            public IEnumerable<Entity> GetDictResources(Guid idService)
            {
                if (DictResources.ContainsKey(idService))
                {
                    return DictResources[idService];
                }
                else return null;

            }

            public void setDictResources(Guid idService, IEnumerable<Entity> lst)
            {
                if(idService != null && lst != null)
                    DictResources[idService] = lst;
            }

            public IEnumerable<Entity> GetDictGroups(Guid idService)
            {
                if (DictGroups.ContainsKey(idService))
                {
                    return DictGroups[idService];
                }
                else return null;

            }
            public void setDictGroups(Guid idService, IEnumerable<Entity> lst)
            {
                if (idService != null && lst != null)
                    DictGroups[idService] = lst;
            }


           
        }



        /// <summary>
        /// // Class model for deserialization result of function webapi Search
        /// </summary>
        public class AnswerTimeSlots
        {
            [JsonProperty("@odata.context")]
            public string odatacontext { get; set; }
            public SearchResult SearchResults { get; set; }

           
        }
        public class SearchResult
        {
            public Proposal[] Proposals { get; set; }
            public TraceInfo TraceInfo { get; set; }        
            public int getProposalsCount()
            {
                if (Proposals == null) return 0;
                int count = 0;
                foreach (Proposal p in Proposals)
                {
                    count++;
                }                
                return count;               
            }
        }

        public class TraceInfo
        {
            public object[] ErrorInfoList { get; set; }
            public int getErrorInfoListCount()
            {
                if (ErrorInfoList == null) return 0;
                int count = 0;
                foreach (object p in ErrorInfoList)
                {
                    count++;
                }
                return count;
            }

        }

        public class Proposal
        {
            public DateTime Start { get; set; }
            public DateTime End { get; set; }
            public string SiteId { get; set; }
            public string SiteName { get; set; }
            public ProposalParty[] ProposalParties { get; set; }
            public int getProposalPartyCount()
            {
                if (ProposalParties == null) return 0;
                int count = 0;
                foreach (ProposalParty p in ProposalParties)
                {
                    count++;
                }
                return count;
            }
        }

        public class ProposalParty
        {
            public string ResourceId { get; set; }
            public string ResourceSpecId { get; set; }
            public string DisplayName { get; set; }
            public string EntityName { get; set; }
            public float EffortRequired { get; set; }
        }





    }

}
